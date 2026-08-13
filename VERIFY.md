# Verifying this record

Four checks, all runnable on a clone without our cooperation.

## 1. Each record hashes its own content

`hash` is the SHA-256 of the record's canonical JSON with `hash` removed —
`json.dumps(payload, sort_keys=True, indent=2, ensure_ascii=False)` plus a
trailing newline, UTF-8.

```python
import hashlib, json, pathlib

rec = json.loads(pathlib.Path("books/best_cagr/snapshots/2026-08-12.json")
                 .read_text(encoding="utf-8"))
body = {k: v for k, v in rec.items() if k != "hash"}
canon = json.dumps(body, sort_keys=True, indent=2, ensure_ascii=False, default=str) + "\n"
assert hashlib.sha256(canon.encode()).hexdigest() == rec["hash"]
```

## 2. The records are chained

Each snapshot's `prev_hash` is the previous session's `hash`; the first is 64
zeroes. `CHAIN.jsonl` records both, append-only.

This is what a timestamp cannot give you: a timestamp proves a file existed, not
that the series is complete. Because each session commits to the one before it, a
day cannot be removed without breaking every record after it.

```bash
python -c "
import json,hashlib,pathlib
prev={}
for line in open('CHAIN.jsonl',encoding='utf-8'):
    e=json.loads(line); p=pathlib.Path(e['file']); raw=p.read_bytes()
    assert hashlib.sha256(raw).hexdigest()==e['sha256'], p
    rec=json.loads(raw.decode())
    assert rec['prev_hash']==prev.get(e['book'],'0'*64), p
    prev[e['book']]=rec['hash']
print('chain ok')
"
```

## 3. The records were not back-dated

```bash
pip install opentimestamps-client
ots verify books/best_cagr/snapshots/2026-08-12.json.ots
```

A fresh proof reads "pending confirmation" until the aggregating Bitcoin
transaction confirms. That means not yet confirmed, not invalid.

## 4. The numbers follow from the inputs

`nav.csv` is the equity curve. Metrics are computed from it by
`compute_core_metrics` in the firm's `rvb.metrics` module, with the risk-free
rate echoed in `metrics.json`. Cumulative return:

```python
import pandas as pd
nav = pd.read_csv("books/best_cagr/nav.csv")
print(nav["equity"].iloc[-1] / nav["equity"].iloc[0] - 1)
```

## What this proves, and what it does not

**Proves:** no published number was edited afterwards, no session was dropped,
each record existed when it claims, and the metrics follow from the equity curve.

**Does not prove:** that the trading was skilful, that these paper fills would
have happened in a real market, or that no other book exists unpublished. Git
history can be rewritten by whoever controls a repository — which is why the hash
chain, the timestamps and branch protection are used together.
