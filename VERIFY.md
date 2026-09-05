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
ots --no-cache verify books/best_cagr/snapshots/2026-08-12.json.ots
```

Two practical notes, because the tooling is fiddly and a check you cannot run is
not a check:

* The proof and the file it commits to must sit **side by side with matching
  names** (`<name>.json` and `<name>.json.ots`), which is how they are published —
  so run the command from inside a clone of this repository, not on a file you
  downloaded on its own.
* Verification asks a Bitcoin block explorer for the block header. If yours is
  unreachable, or the client's bundled explorer list has moved on, `ots verify`
  reports that it could not confirm rather than that the proof is bad. Point it
  at a node you trust with `ots --bitcoin-node <url> verify …`, or read the
  proof offline with `ots info <file>.ots` and check the block height it names
  against any explorer by hand.

A fresh proof is a commitment to a calendar server and is *incomplete* until the
aggregating Bitcoin transaction confirms — normally within a few hours. Run
`ots upgrade <file>.ots` to complete it (the publisher does this automatically on
each run). An incomplete proof means "not yet confirmed", not "invalid".

### What a timestamp does NOT prove

A proof establishes that a file existed **no later than** its block. It says
nothing about how much earlier, so it bounds back-dating in one direction only:
it cannot show a record was written on the day it describes.

That distinction is not academic here. Where a book's chain was rebuilt, its
proofs were all created in the **same batch on the day of the rebuild** — so
sixteen records covering sessions from 12 to 27 August share one anchoring
date, and none of them corroborates its own session.

The lag is not hidden: every `CHAIN.jsonl` entry already carries `session_date`
(the day it describes) and `ts` (the moment it was appended), so measure it
yourself for every record ever published:

```python
import datetime, json
for e in (json.loads(l) for l in open("CHAIN.jsonl", encoding="utf-8") if l.strip()):
    session = datetime.date.fromisoformat(e["session_date"])
    appended = datetime.datetime.fromisoformat(e["ts"]).date()
    print(e["book"], e["session_date"], (appended - session).days, "days to chain")
```

A record chained the day after its session is routine. A run of records sharing
one `ts` is a rebuild, and you should read them as one act rather than as
independent daily evidence.

Read together with section 2, this still forbids the thing that matters: the
chain makes a session impossible to drop later without breaking every record
after it. What it does not do is date a record on its own.

## 4. The numbers follow from the inputs

`books/<book>/nav.csv` is the whole equity curve. Every published metric is
computed from it by `compute_core_metrics` in the firm's `rvb.metrics` module,
with `risk_free_annual` set to the value echoed in `metrics.json`. Recompute and
compare.

**Use `equity_adj`, not `equity`.** The file carries both on purpose: `equity` is
the broker's raw figure, never rewritten, and `equity_adj` is that figure with
declared external capital movements taken out of the return (`flow` and
`adj_factor` are published beside them so you can redo the adjustment yourself).
On a book with no declared event the two columns are identical. On a book with
one — `best_cagr` has two — the raw column will NOT reproduce the published
cumulative return, and that difference is the adjustment working, not an error.

```python
import pandas as pd
nav = pd.read_csv("books/best_cagr/nav.csv")

# the published cumulative return: capital movements removed
print(nav["equity_adj"].iloc[-1] / nav["equity_adj"].iloc[0] - 1)

# the raw broker equity, for comparison — differs whenever a flow was declared
print(nav["equity"].iloc[-1] / nav["equity"].iloc[0] - 1)
```

Every declared movement is published with its evidence inside the snapshot for
the session it hit; `meta.json` lists them under `capital_events`.

### What a passing chain does not cover

The chain proves that the records **in it** are complete and unaltered. It does
not, by itself, tell you that a book's chain was ever *restarted*. One has been:
on 2026-08-27 the real-capital book's entire prior record was set aside after its
accounting convention was found to be wrong at the sign level — the old records
verify exactly as they always did, they were simply wrong about the world, which
no hash can detect. They are kept, unedited, at
`books/maker_01/superseded/`, with a written explanation of what was wrong and
why the replacement is different, and the live chain for that book starts again
from a genesis entry. A restart is therefore always visible as a second chain on
disk — but you have to look for it, so it is stated here rather than left for you
to find.

## 5. Against the broker

Each **paper-account** snapshot's `broker_cross_check` carries Alpaca's own daily equity for the
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

Two kinds of artefact are recorded, and **both are checkable**. `detail` is the
order/fill file, hashed as its own bytes. `intraday` is one session's slice of
the accumulated 5-minute curve, hashed as the CSV that slice serialises to:
the session's rows sorted by `timestamp`, written with a header, no index, and
`\n` line endings — i.e. exactly `chunk.to_csv(index=False, lineterminator="\n")`.
Stating that convention is what makes the 5-minute curve verifiable at all; it
cannot be recomputed from the snapshots the way `nav.csv` can.

```python
import hashlib, json, pathlib
import pandas as pd

chain = [json.loads(l) for l in open("DETAIL_CHAIN.jsonl", encoding="utf-8") if l.strip()]
prev = "0" * 64
for e in chain:                                   # links intact
    assert e["prev_hash"] == prev, e
    prev = e["hash"]

# Only the NEWEST entry per (book, session, kind) matches the file on disk:
# a correction appends, so older entries describe superseded versions.
latest = {(e["book"], e["session_date"], e["kind"]): e["sha256"] for e in chain}

checked = {"detail": 0, "intraday": 0}
for (book, session, kind), digest in latest.items():
    if kind == "detail":                          # the orders and fills
        raw = pathlib.Path(f"books/{book}/detail/{session}.json").read_bytes()
        assert hashlib.sha256(raw).hexdigest() == digest, (book, session, kind)
    elif kind == "intraday":                      # that session's 5-minute rows
        curve = pd.read_csv(f"books/{book}/intraday.csv")
        chunk = curve[curve["session_date"].astype(str) == session]
        raw = chunk.sort_values("timestamp").to_csv(
            index=False, lineterminator="\n").encode("utf-8")
        assert hashlib.sha256(raw).hexdigest() == digest, (book, session, kind)
    else:
        raise AssertionError(f"unknown artefact kind: {kind}")
    checked[kind] += 1
print(checked)                                    # nothing silently skipped
```

The final `else` matters more than it looks. An earlier version of this script
tested only `kind == "detail"` and said nothing about the rest, so it reported
success while silently skipping 96 of 179 entries. A verification script that
can pass without checking anything is worse than none.

A correction appends; it never overwrites. So a session whose detail changed
appears twice, with two timestamps, and only the newest entry matches the file
on disk.

### A superseded chain

Where a book's chain was rebuilt, the previous one is kept verbatim under
`books/<book>/superseded/` — nothing is deleted. Its `CHAIN.jsonl` records the
paths the files had **when they were published**, i.e. `books/<book>/snapshots/...`,
and those paths are now occupied by the replacement chain. Verify the archive
against the archive:

```python
import hashlib, json, pathlib
base = pathlib.Path("books/<book>/superseded")
prev = "0" * 64
for line in (base / "CHAIN.jsonl").read_text(encoding="utf-8").splitlines():
    if not line.strip():
        continue
    e = json.loads(line)
    assert e["prev_hash"] == prev, e
    prev = e["hash"]
    # The recorded `file` is the ORIGINAL path; the archived copy keeps only
    # the file name.
    raw = (base / "snapshots" / pathlib.Path(e["file"]).name).read_bytes()
    assert hashlib.sha256(raw).hexdigest() == e["sha256"], e["session_date"]
```

Following the recorded paths literally verifies the *replacement* files and
fails on every entry — which reads as a broken archive when the archive is
intact. What this forbids is the quiet substitution: publishing one set of fills
and later serving another with nothing to show for it.

## 7. Capital events, if a book has one

A capital movement that is not a trade is excluded from the return and kept in
the balance (Methodology, section 2). Because that is an adjustment we make to
our own performance figure, it is the one thing in this repository that most
deserves checking, so it is published to be checked.

**Only a book that can take a capital movement carries these columns.** The
paper books do; `maker_01` has a `nav.csv` of `date,equity,cash,daily_return`
and no `flow`/`adj_factor`/`equity_adj` — but that is **not** because it takes
no deposits. It does. Its collector removes them by **unitisation**: a deposit
buys units at the day's price, so it moves the balance and never the price, and
`daily_return` is already the unit return. The flow adjustment the paper books
apply after the fact is done at source there instead.

So on that book `equity` is a BALANCE, not the index, and rebasing it will not
reproduce the headline. Check for the columns before running the script below:

```python
import pandas as pd
nav = pd.read_csv("books/<book>/nav.csv")
adjusted = {"flow", "adj_factor", "equity_adj"} <= set(nav.columns)
```

For an adjusted book, `nav.csv` carries both series and the bridge between them:

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
dated and unable to reach backwards; each record existed **no later than** the block its proof is anchored in (`.ots`) — section 3 shows how to measure the lag between a session and its proof, and why a batch of records sharing one anchor is one act rather than daily evidence;
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
