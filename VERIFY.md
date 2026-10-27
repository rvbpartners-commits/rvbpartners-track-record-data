# How to verify this track record

Everything below can be done by a stranger with a clone of this repository and no
cooperation from us. Run each script from the root of the clone.

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

snapshots = sorted(pathlib.Path("books").glob("*/snapshots/*.json"))
for path in snapshots:
    rec = json.loads(path.read_text(encoding="utf-8"))
    body = {k: v for k, v in rec.items() if k != "hash"}
    assert hashlib.sha256(canonical(body).encode()).hexdigest() == rec["hash"], path
print(len(snapshots), "records hash their own content")
```

Change any published number and this fails.

## 2. The records are chained

Each snapshot's `prev_hash` is the `hash` of that book's previous session. The
first is 64 zeroes; for each paper book that first record is its base, dated
2026-10-16. `CHAIN.jsonl` records, append-only, one line per snapshot with its
file path, its SHA-256 as bytes on disk, its `prev_hash` and its `hash`.

This is what a timestamp alone cannot give you. OpenTimestamps proves a file
existed; it says nothing about whether the *series* is complete. Because each
session commits to the one before it, a day cannot be removed later without
breaking every record after it — so selective omission, the failure mode that
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

Each snapshot is stamped with OpenTimestamps when it is published. A proof
anchors — gains its Bitcoin block — normally within hours; until then the
verifier lists it as pending, and a proof still pending after seven days fails
verification outright. Pick any record in `books/<book>/snapshots/` and verify
its proof:

```bash
pip install opentimestamps-client
ots --no-cache verify books/<book>/snapshots/<date>.json.ots
```

Two practical notes:

* The proof and the file it commits to must sit side by side with matching
  names (`<name>.json` and `<name>.json.ots`), which is how they are published —
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
`ots upgrade <file>.ots` to complete it (each book's publisher does this on its
runs; every proof of the closed `maker_01` is complete). An incomplete proof means
"not yet confirmed", not "invalid".

### What a timestamp does NOT prove

A proof establishes that a file existed no later than its block. It says
nothing about how much earlier, so it bounds back-dating in one direction only:
it cannot show a record was written on the day it describes. Records chained in
one act share one anchoring date, and none of them corroborates its own session.

The lag is measurable: every `CHAIN.jsonl` entry carries `session_date` (the day
it describes) and `ts` (the moment it was appended), so measure it for every
record:

```python
import datetime, json
for e in (json.loads(l) for l in open("CHAIN.jsonl", encoding="utf-8") if l.strip()):
    session = datetime.date.fromisoformat(e["session_date"])
    appended = datetime.datetime.fromisoformat(e["ts"]).date()
    print(e["book"], e["session_date"], (appended - session).days, "days to chain")
```

A record chained the day after its session is routine. A run of records sharing
one `ts` was chained in one act, and reads as one act rather than as independent
daily evidence.

Read together with section 2, this still forbids the thing that matters: the
chain makes a session impossible to drop later without breaking every record
after it. What it does not do is date a record on its own.

## 4. The numbers follow from the inputs

`books/<book>/nav.csv` is the whole equity curve, and every published metric
follows from it under the conventions METHODOLOGY section 3 states. The firm's
metrics module is not published; recompute with your own implementation of those
conventions, at the `risk_free_annual` echoed beside each figure, and compare. The
one convention a textbook implementation does not share: the maximum drawdown and
the Calmar ratios are measured with a zero return prepended, so the record's first
value is the first high-water mark.

**Use `equity_adj`, not `equity`.** The file carries both on purpose: `equity` is
the broker's raw figure, never rewritten, and `equity_adj` is that figure with
declared external capital movements taken out of the return (`flow` and
`adj_factor` are published beside them so you can redo the adjustment yourself).
On a book with no declared event the two columns are identical. On a book with
one, the raw column does not reproduce the published cumulative return, and the
difference is the adjustment.

```python
import pandas as pd

book = "best_cagr"                      # any book listed in index.json
nav = pd.read_csv(f"books/{book}/nav.csv")

# the published cumulative return: capital movements removed
print(nav["equity_adj"].iloc[-1] / nav["equity_adj"].iloc[0] - 1)

# the raw broker equity, for comparison — differs whenever a flow was declared
print(nav["equity"].iloc[-1] / nav["equity"].iloc[0] - 1)

# the maximum drawdown, base included (published once a book has 60 sessions)
path = nav["equity_adj"] / nav["equity_adj"].iloc[0]
print((path / path.cummax() - 1).min())
```

Every declared movement is published with its evidence inside the snapshot for
the session it hit; `meta.json` lists them under `capital_events`.

### What a passing chain does not cover

The chain proves that the records in it are complete and unaltered. It cannot
show that a record was right about the world: a record written under a wrong
accounting convention verifies exactly like a correct one. `maker_01`'s earlier
chains are kept unedited under `books/maker_01/superseded/`, with a written
explanation of what each got wrong and why its replacement differs; the live
chain for that book starts from its own genesis entry. Section 6 shows how to
verify those archives.

## 5. Against the broker

Each paper-account snapshot's `broker_cross_check` carries Alpaca's own daily
portfolio-history equity beside our published NAV, and the difference in basis
points; both are broker figures and neither is adjusted to match the other. What a
record compares is stated in the record itself (METHODOLOGY section 6):

- A record without `basis` in that block — the first records, from each paper
  book's base record of 16 October 2026 — compares its session's NAV
  (`nav.equity`) with the broker's figure for the previous session. The broker's
  timestamp for a session's figure falls on the following day in UTC; those records
  read it by that date. The 2026-10-21 record's figure is therefore the 2026-10-20
  close, and the Monday records carry none.
- A record with `basis` compares the session it names (`session_date`, whose
  NAV it repeats as `nav_equity`) with the broker's figure for that same session:
  its own session when the broker had served that figure when the record was
  written, and otherwise the one before.

Check either kind against the NAV the record names:

```python
import json, pathlib

for path in sorted(pathlib.Path("books").glob("*/snapshots/*.json")):
    rec = json.loads(path.read_text(encoding="utf-8"))
    check = rec.get("broker_cross_check") or {}
    ours = check.get("nav_equity", (rec.get("nav") or {}).get("equity"))
    theirs = check.get("portfolio_history_equity")
    if ours is None or not theirs:
        continue
    assert abs(round((ours / theirs - 1) * 10_000, 2) - check["delta_bps"]) < 0.011, path
```

## 6. The trades, and the intraday curve

Snapshots are write-once: a second version is a restatement and the publisher
refuses. Execution detail is deliberately *correctable*, and the 5-minute series
is accumulated rather than derived, so neither can live in the write-once chain.

`DETAIL_CHAIN.jsonl` is where they are recorded instead: append-only, each entry
linked to the last, one entry per version of each artefact.

Two kinds of artefact are recorded, and both are checkable. `detail` is the
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

The final `else` is deliberate: an entry of a kind the script does not know
stops it, so the script cannot report success on entries it never checked.

A correction appends; it never overwrites. So a session whose detail changed
appears twice, with two timestamps, and only the newest entry matches the file
on disk. What this forbids is the quiet substitution: publishing one set of fills
and later serving another with nothing to show for it. The six `detail` entries of
2026-10-24 09:50 UTC revise each paper book's 2026-10-22 file in one field only,
`open_pnl_pct`, from its fifth to its eighth significant digit (by at most 2.6e-5 of
its value): the ratio had been computed from the open P&L before it was rounded to
cents, and is now computed from the published figure.

### A superseded chain

`maker_01` keeps its earlier chains under `books/maker_01/superseded/`: one in
that folder itself and any later one in a named subfolder, each with its own
`CHAIN.jsonl` starting from 64 zeroes. Each `CHAIN.jsonl` records the paths the
files had when they were published, i.e. `books/maker_01/snapshots/...`, and
those paths are now occupied by the live chain. Verify each archive against
itself:

```python
import hashlib, json, pathlib

books = pathlib.Path("books")
archives = [*books.glob("*/superseded/CHAIN.jsonl"),
            *books.glob("*/superseded/*/CHAIN.jsonl")]
for chain in sorted(archives):
    prev = "0" * 64
    for line in chain.read_text(encoding="utf-8").splitlines():
        if not line.strip():
            continue
        e = json.loads(line)
        assert e["prev_hash"] == prev, (chain, e["session_date"])
        prev = e["hash"]
        # The recorded `file` is the ORIGINAL path; the archived copy keeps only
        # the file name.
        raw = (chain.parent / "snapshots" / pathlib.Path(e["file"]).name).read_bytes()
        assert hashlib.sha256(raw).hexdigest() == e["sha256"], (chain, e["session_date"])
    print(chain.parent, "ok")
```

Following the recorded paths literally verifies the *live* files and fails on
every entry, which reads as a broken archive when the archive is intact.

## 7. Capital events, if a book has one

A capital movement that is not a trade is excluded from the return and kept in
the balance (Methodology, section 2). Because that is an adjustment we make to
our own performance figure, it is the one thing in this repository that most
deserves checking, so it is published to be checked.

**Every book carries these columns, filled in one of two ways.** The paper books
adjust after the fact, as described below. `maker_01`'s collector removes
movements at source by unitisation: a deposit buys units at the day's price,
so it moves `equity` and never the unit value. On that book `equity_adj` is the
unit value and `adj_factor` is `equity_adj / equity`, so check 2 below, which
reproduces the paper books' factor, does not apply to it. Checks 1 and 3 do;
its values are published to six decimals, so run check 3 on it with a tolerance
of `1e-9`.

`nav.csv` carries both series and the bridge between them:

| column | what it is |
|---|---|
| `equity` | exactly what the broker reported. Never rewritten. |
| `flow` | the declared external movement on that date, signed. Zero almost everywhere. |
| `adj_factor` | the multiplier that removes flows from the return. |
| `equity_adj` | `equity x adj_factor` — the track-record index, and what the charts draw. On `maker_01`, the unit value. |

Three checks, all of which a reader can run against files in this repository:

```python
import pandas as pd

book = "best_cagr"                      # any book listed in index.json
nav = pd.read_csv(f"books/{book}/nav.csv")

# 1. The adjustment cannot reach backwards. Before the first declared flow,
#    the adjusted index IS the raw equity, to the cent.
flagged = nav.index[nav["flow"] != 0]
before = nav.iloc[:flagged.min()] if len(flagged) else nav
assert (before["adj_factor"] == 1.0).all()
assert (before["equity_adj"] - before["equity"]).abs().max() < 0.01

# 2. The factor is what it claims to be (paper books):
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
it is hash-chained and timestamped like every other claim here, and it
carries a `derivation` field stating how the amount was obtained from published
data rather than merely asserting it.

## What this does and does not prove

**It proves:** no published number has been edited after the fact; no session has
been quietly dropped; every adjustment to a return is declared, evidenced, dated
and unable to reach backwards; each record whose proof has anchored existed
**no later than** its Bitcoin block — the newest records may still be pending
their block, the verifier says which, and none may stay pending beyond seven
days (section 3 shows how to check a proof and how to measure the lag between
a session and its chain entry); the orders, fills and
intraday curve match the digests recorded for them, and any correction to them is
visible; the metrics follow from the equity curve under the conventions
METHODOLOGY.md states, so they can be recomputed independently.

**It does not prove:** that the trading itself was skilful, that the paper fills
would have happened in a real market, or that we are not running other,
unpublished books. Git history can be rewritten by whoever controls the
repository — which is exactly why the hash chain, the OpenTimestamps proofs and
the branch ruleset on `main` (no deletion, no force-push, linear history,
signatures required) are used together rather than relying on any one of them.
Every publish commit is signed; `git log --show-signature` checks that against
the key, and GitHub shows it as Verified.

**Nor does it say anything about whether the results are good.** The record is
published so that it can be checked.
