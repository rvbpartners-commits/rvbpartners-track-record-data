# How to verify this track record

Everything below can be done by a stranger with a clone of this repository and no
cooperation from us.

## 1. Each record hashes its own content

Every file in `books/<book>/snapshots/` carries a `hash` field: the SHA-256 of
the record's canonical JSON with the `hash` field itself removed. Canonical JSON
here means `json.dumps(payload, sort_keys=True, indent=2, ensure_ascii=False)`
plus a trailing newline, encoded UTF-8.

```python
import hashlib, json, pathlib

def canonical(payload):
    return json.dumps(payload, sort_keys=True, indent=2,
                      ensure_ascii=False, default=str) + "\n"

rec = json.loads(pathlib.Path("books/best_cagr/snapshots/2026-08-12.json")
                 .read_text(encoding="utf-8"))
body = {k: v for k, v in rec.items() if k != "hash"}
assert hashlib.sha256(canonical(body).encode()).hexdigest() == rec["hash"]
```

Change any published number and this fails.

## 2. The records are chained

Each snapshot's `prev_hash` is the `hash` of that book's previous session. The
first is 64 zeroes. `CHAIN.jsonl` records, append-only, one line per snapshot
with its file path, its SHA-256 as bytes on disk, its `prev_hash` and its `hash`.

This is what a timestamp alone cannot give you. OpenTimestamps proves a file
existed; it says nothing about whether the *series* is complete. Because each
session commits to the one before it, **a day cannot be removed later without
breaking every record after it** — so selective omission, the failure mode that
actually matters for a track record, leaves evidence.

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
print('chain ok:', {k:v[:12] for k,v in prev.items()})
"
```

## 3. The records were not back-dated

Each snapshot has an OpenTimestamps proof beside it.

```bash
pip install opentimestamps-client
ots verify books/best_cagr/snapshots/2026-08-12.json.ots
```

A fresh proof is a commitment to a calendar server and is *incomplete* until the
aggregating Bitcoin transaction confirms — normally within a few hours. Run
`ots upgrade <file>.ots` to complete it (the publisher does this automatically on
each run). An incomplete proof means "not yet confirmed", not "invalid".

## 4. The numbers follow from the inputs

`books/<book>/nav.csv` is the whole equity curve. Every published metric is
computed from it by `compute_core_metrics` in the firm's `rvb.metrics` module,
with `risk_free_annual` set to the value echoed in `metrics.json`. Recompute and
compare. Cumulative return is a one-liner:

```python
import pandas as pd
nav = pd.read_csv("books/best_cagr/nav.csv")
print(nav["equity"].iloc[-1] / nav["equity"].iloc[0] - 1)
```

## 5. Against the broker

Each snapshot's `broker_cross_check` carries Alpaca's own daily equity for the
same date and the difference from our published NAV in basis points. The two
endpoints have different timing bases and are expected to differ slightly; both
are broker figures and neither is adjusted to match the other.

## 6. The trades, and the intraday curve

Snapshots are write-once: a second version is a restatement and the publisher
refuses. Execution detail is deliberately *correctable* (the first release keyed
positions by strategy slug, and withdrawing that was worth a rewrite), and the
5-minute series is accumulated rather than derived, so neither can live in the
write-once chain.

`DETAIL_CHAIN.jsonl` is where they are recorded instead: append-only, each entry
linked to the last, one entry per version of each artefact.

```python
import hashlib, json, pathlib
chain = [json.loads(l) for l in open("DETAIL_CHAIN.jsonl", encoding="utf-8") if l.strip()]
prev = "0" * 64
for e in chain:                                   # links intact
    body = {k: v for k, v in e.items() if k != "hash"}
    assert e["prev_hash"] == prev
    prev = e["hash"]
latest = {(e["book"], e["session_date"], e["kind"]): e["sha256"] for e in chain}
for (book, session, kind), digest in latest.items():
    if kind == "detail":                          # the orders and fills
        raw = pathlib.Path(f"books/{book}/detail/{session}.json").read_bytes()
        assert hashlib.sha256(raw).hexdigest() == digest, (book, session)
```

A correction appends; it never overwrites. So a session whose detail changed
appears twice, with two timestamps, and only the newest entry matches the file
on disk. What this forbids is the quiet substitution: publishing one set of fills
and later serving another with nothing to show for it.

## 7. Capital events, if a book has one

A capital movement that is not a trade is excluded from the return and kept in
the balance (Methodology, section 2). Because that is an adjustment we make to
our own performance figure, it is the one thing in this repository that most
deserves checking, so it is published to be checked.

`nav.csv` carries both series and the bridge between them:

| column | what it is |
|---|---|
| `equity` | exactly what the broker reported. Never rewritten. |
| `flow` | the declared external movement on that date, signed. Zero almost everywhere. |
| `adj_factor` | the multiplier that removes flows from the return. |
| `equity_adj` | `equity x adj_factor` — the track-record index, and what the charts draw. |

Three checks, all of which a reader can run against files in this repository:

```python
import pandas as pd

nav = pd.read_csv("books/best_cagr/nav.csv")

# 1. The adjustment cannot reach backwards. Before the first declared flow,
#    the adjusted index IS the raw equity, to the cent.
flagged = nav.index[nav["flow"] != 0]
before = nav.iloc[:flagged.min()] if len(flagged) else nav
assert (before["adj_factor"] == 1.0).all()
assert (before["equity_adj"] - before["equity"]).abs().max() < 0.01

# 2. The factor is what it claims to be:
#    k_t = k_{t-1} * E_{t-1} / (E_{t-1} + F_t)
k = 1.0
for i in range(1, len(nav)):
    f = nav["flow"].iloc[i]
    if f:
        k *= nav["equity"].iloc[i - 1] / (nav["equity"].iloc[i - 1] + f)
    assert abs(nav["adj_factor"].iloc[i] - k) < 1e-12

# 3. daily_return is the pct_change of the adjusted index, not the raw one.
assert (nav["equity_adj"].pct_change() - nav["daily_return"]).abs().max() < 1e-12
```

The event itself, with its evidence, is inside the write-once snapshot for the
session it hit — `books/<book>/snapshots/<date>.json`, key `capital_event` — so
it is hash-chained and Bitcoin-anchored like every other claim here, and it
carries a `derivation` field stating how the amount was obtained from published
data rather than merely asserting it.

## What this does and does not prove

**It proves:** no published number has been edited after the fact; no session has
been quietly dropped; every adjustment to a return is declared, evidenced,
dated and unable to reach backwards; each record existed when it claims to (`desk publish
--verify` opens every proof, checks it commits to that exact file, and reports
the Bitcoin block it is anchored in); the orders, fills and intraday curve match
the digests recorded for them, and any correction to them is visible; the metrics
follow from the equity curve by open code.

**It does not prove:** that the trading itself was skilful, that the paper fills
would have happened in a real market, or that we are not running other,
unpublished books. Git history can be rewritten by whoever controls the
repository — which is exactly why the hash chain, the OpenTimestamps proofs and
the branch ruleset on `main` (no deletion, no force-push, linear history,
signatures required) are used together rather than relying on any one of them.
Every publish commit is signed; `git log --show-signature` checks that against
the key, and GitHub shows it as Verified.

**And it says nothing about the results being good.** These are paper accounts
with a short history. The point of this repository is that the record is
checkable, not that it is impressive.
