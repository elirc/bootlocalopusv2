# Verification gap: why there is no worked example here (yet)

The course convention (inherited from the rebuild course) is that this folder
holds one break-the-guarantee experiment, actually performed, with real
captured output — and that the experiment's baseline must be fully green
first. On the day this course was written (2026-10-01, Windows 11, Node 22,
fresh `npm ci`), the baseline was not fully green on the authoring machine, so
the honest deliverable is this document instead. Everything quoted below is
verbatim from runs that actually happened; nothing is reconstructed.

## What ran green

`npm run typecheck` — clean (no diagnostics, exit 0).

`npm run test:unit` (the economy's own suite):

```
# tests 31
# suites 8
# pass 31
# fail 0
```

## What did not

`npm run test:sandbox` (the grader's guard-rail suite, 43 checks), twice.
First run, tail:

```
FAIL [mutation: react subject] ok=false 60403ms (boot -ms)
       error: The grader took too long to start (this is not your code). Try again.
       !! ok=false, expected true
       !! 0 rows, expected 4
       !! score=undefined, expected 2/2

6 of 43 checks failed
```

Second run (failures only):

```
FAIL [react hook + event] ok=false 60485ms (boot -ms)
       error: The grader took too long to start (this is not your code). Try again.
       !! ok=false, expected true
FAIL [react: dispatchEvent(new Event(x)) uses the DOM realm; storage resets between tests] ok=false 60383ms (boot -ms)
       error: The grader took too long to start (this is not your code). Try again.
       !! ok=false, expected true
       !! 0 rows, expected 3
FAIL [mutation: react subject] ok=false 60255ms (boot -ms)
       error: The grader took too long to start (this is not your code). Try again.
       !! ok=false, expected true
       !! 0 rows, expected 4
       !! score=undefined, expected 2/2
3 of 43 checks failed
```

Then the first failing case alone, on an otherwise idle run:

```
$ npx tsx scripts/smoke.ts "react hook"
ok   [react hook + event] ok=true 31788ms (boot 29867ms)

all 1 checks passed
```

`npm run test:api` was started and then stopped by the host because the
machine ran critically low on memory; no result exists for it from this
session, so none is claimed.

## Reading the failure honestly

Every failing check is a react-environment case, every failure is the same
mode — the worker's **boot** (importing jsdom + React DOM + the harness)
exceeding the 60 s boot budget (`BOOT_BUDGET_MS`,
`server/runner/session.ts:32`) — and the failing set shrank from 6 to 3
between the two full runs while the same case passed in 31.8 s run alone.
That pattern says load flake, not code defect: under a full-suite run on this
machine (with everything else this session was doing), jsdom's cold import
sometimes crosses 60 s; alone, it does not.

Two things are worth noticing before dismissing this as "just a slow laptop":

1. **The engine already behaves correctly about its own flakiness.** The
   error text — *"The grader took too long to start (this is not your
   code)"* — comes from `abnormalEnd` (`session.ts:316`), and `ready: false`
   means `isScored` (`server/app.ts:121`) refuses to count the run as an
   attempt. A learner hitting this exact flake loses nothing but time. The
   smoke suite goes red because for *itself* the engine accepts no excuses:
   a case that expected `ok=true` did not get it, whatever the reason. The
   suite and the product disagree about whose fault a slow boot is, and both
   are right for their audience — that distinction is the lesson.
2. **"43 checks, 3 failed" is a statement about this machine on this day**,
   with this session's memory pressure, not about the grader's logic. The
   isolated green run is evidence the logic is fine; it is *not* evidence the
   suite would be green end to end here. Those are different claims, and this
   file refuses to trade one for the other.

## What closing the gap looks like

On a machine (or a quiet moment) where the four engine commands pass clean —

```powershell
npm run typecheck; npm run test:unit; npm run test:sandbox; npm run test:api
```

— perform EXERCISES.md Tier 1.1 (the pass-condition break in
`server/runner/index.ts:100`) in the worked-example format the rebuild course
models (predict → one-line break → both runs' real output → revert → green
rerun), and replace this file with it. If your first break turns out to be
absorbed by another layer, keep going one layer down and report both stages —
Stockade's oversell experiment is the model for that shape. Until then, this
gap report *is* the worked example: the thing being demonstrated is that you
do not publish a run you did not get.
