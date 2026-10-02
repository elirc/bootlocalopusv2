# Exercises: against the engine

Reading a grader teaches less than breaking one and watching which layer
notices. These are ordered by tier; each has a **done when** anchored to real
files and commands. Work on a branch; write your prediction down before every
run; revert deliberate breaks with `git checkout -- <file>` and never commit
one.

Baseline for everything below (the engine's own suite, not the lessons):

```powershell
npm run typecheck; npm run test:unit; npm run test:sandbox; npm run test:api
```

All four must pass before you start and after you finish each exercise.
(`npm test` additionally runs `npm run verify`, which grades all 508 lessons'
references — correct, but slow; use `npm run verify -- --changed` while
iterating.)

---

## Tier 1 — Break and observe (half a day each)

The smoke suite (`scripts/smoke.ts`) is the engine's conscience: every case
states what the grader *must conclude* about a known input, including hostile
ones. Break a guarantee and predict which cases go red and what a learner at
the UI would have experienced if the break had shipped.

**1.1 Break the pass condition.** In `server/runner/index.ts:100`, change
`tests.every((t) => t.passed === true)` to `tests.some((t) => t.passed === true)`.
*Predict first:* which smoke cases fail — only the "wrong code must fail"
basics, or the forgery cases too? What would a learner experience: free XP on
every lesson, or only on lessons with at least one soft assertion?
*Done when:* you can name the failing labels from `npm run test:sandbox`
without re-reading `smoke.ts`, and explain why this one line is the entire
verdict and why the worker is not allowed a vote.

[worked-examples/](worked-examples/) is reserved for a completed run of this
break with real captured output; today it holds
[VERIFICATION-GAP.md](worked-examples/VERIFICATION-GAP.md) — read it for why,
and for the standard your own run must meet before it may replace that file.

**1.2 Break the timeout handling.** In `server/runner/session.ts`, delete the
`armBudget()` call inside the `ready` handler (`:240`), so no learner budget is
ever armed after boot.
*Predict first:* the infinite-loop smoke case (`export const spin = () => {
while (true) {} }`, budget 4 s) — does it now hang forever, or end at some
other limit? Which limit, after how long, and with what phase? Would the run
be scored against the learner (`isScored`, `server/app.ts:121`)?
*Done when:* you observed which *other* clock caught the loop and can explain
the two-clock design (boot budget vs learner budget) and what each one owes
the learner.

**1.3 Break progress persistence.** In `server/progress.ts` (`writeAtomic`,
`:223`), replace the write-fsync-rename dance with a plain
`writeFile(target, contents)`.
*Predict first:* does any suite fail? (`npm run test:api` exercises the API
against a scratch `DATA_DIR`.) If everything stays green, what did you just
learn about the difference between "tested" and "safe"? What real-world event
is the fsync+rename protecting against, and how would a learner experience its
absence?
*Done when:* you can say which part of the durability promise is pinned by a
test and which part is only pinned by review — and you have checked, not
guessed. If you find it is review-only, that is a finding worth a journal
entry, not a shrug.

**1.4 Break the trust boundary, as the learner.** No engine edit: write a
"solution" for any `js` lesson that tries to forge a pass. Try the three v1
forgeries from `upskilling/journal/005-*.md` (reassign `expect`, post to
`parentPort`, patch `Array.prototype.every`) and one of your own.
*Predict first:* for each, which guard stops it and with what message?
*Done when:* each attempt fails a different way and you can point at the
specific line in `guard.mjs` / `primordials.mjs` / `session.ts` that stopped
it — and you have also written down one thing a learner *can* still do (§7 of
the code tour) that no guard stops.

## Tier 2 — Extend inside the boundaries (1–3 days each)

**2.1 Author a graded lesson end to end.** Pick a gap you noticed as a
learner and write a real lesson in the real format: a folder under the right
chapter with `brief.md`, `starter.*`, `solution.*`, `tests.*`, plus its typed
metadata in `chapter.ts` (walk `content/AUTHORING.md` first — id equals folder
name, nothing else allowed in the folder).
*Done when:* `npm run lesson <id>` passes the reference and
`npm run lesson <id> -- --starter` fails it *fast and for the stated reason*;
`npm run verify -- --changed` is green; and `npm run mutate -- --lesson=<id>`
shows your tests catch at least the obvious mutants. The mutation check is the
senior half of the exercise: a lesson whose tests pass a broken solution is
worse than no lesson.

**2.2 Grader determinism, measured.** The engine works to make verdicts
deterministic (serialised gradings, boot excluded, per-test timeouts) — now
measure it. Write a scratch script (do not commit it) that calls
`runExercise` from `server/runner/index.ts` twenty times with the same input —
e.g. the `node-http-router` reference against its own tests — and compares the
results with `ms`/`bootMs` stripped.
*Predict first:* identical 20/20? If not, which field differs and why?
*Done when:* you have 20 recorded outcomes and a one-paragraph statement of
the result at its actual strength ("identical on this machine on this day
under no load" is the honest ceiling — say that), plus, if any run differed,
the layer you believe is responsible and the experiment that would confirm it.

**2.3 Corruption recovery, rehearsed.** On a scratch data dir
(`$env:DATA_DIR = "...\scratch-data"`), run the server, pass a lesson, stop
it, then truncate `progress.json` mid-document. Restart.
*Predict first:* what does the server log, where does the corrupt file go,
what state does the UI show? Then delete the backups directory too and repeat.
*Done when:* you have seen both recovery paths in `server/progress.ts`
(`quarantineAndRecover`, `:202`) fire for real — quarantine + newest backup,
and quarantine + fresh start — and you can say what is *lost* in each case and
whether the loss is bounded by anything stronger than the daily backup cadence.

## Tier 3 — Move a boundary (1–2 weeks each)

One page of design first: two alternatives, chosen tradeoffs, what evidence
would make you revisit. Build under the existing suites; update the journal as
if the next learner inherits your version. These are **exercises** — none of
this exists in the engine today.

**3.1 True isolation: process-level sandbox.** `guard.mjs` says it plainly:
the worker thread is not a security boundary. Design and build one grading
kind running in a separate OS process (or container) with: no filesystem
access outside the run dir (not just no *imports* — no `fs` reads), no
network, a real CPU/RSS limit, and the same streamed-rows protocol.
*Done when:* every current smoke case for that kind still concludes the same
verdicts; a new smoke case proves `fs.readFileSync('<outside path>')` fails
inside graded code while the `node` lesson's real sockets still work (or you
document why those two demands forced you to split kinds); and the code tour's
§7 can be rewritten one honest notch stronger — with the latency cost of the
process spawn measured and stated next to it.

**3.2 Multi-user progress with versioned writes.** `data/progress.json` is a
single-player cache-and-save with a process lock. Make the store safe for two
concurrent writers (two browser profiles, or classroom mode from
`../ROADMAP.md` mission 22) the way the rebuild course does it: a version (or
ETag) on the progress document, compare-and-swap on write, 409 with current
state on conflict. GitJira's CODE-TOUR and its `If-Match` discipline — and
this curriculum's own `node-conditional-updates` / `node-db-optimistic-locking`
lessons — are the design background.
*Done when:* a two-process test in the spirit of the rebuilds' concurrency
suites proves exactly one of two racing submits pays XP and the loser gets a
conflict, not a silent overwrite; and the single-user path still passes
`npm run test:api` unchanged.

**3.3 Grade the rebuild course inside this engine.** The capstone track
([CAPSTONE-TRACK.md](CAPSTONE-TRACK.md)) currently grades itself with each
rebuild's own `npm test`. Design a lesson kind (or a `node-db` chapter) that
embeds a rebuild exercise as a first-class graded lesson here — the natural
first candidate: GitJira's deferred **"port to Postgres under acceptance
tests"** (its EXERCISES.md Tier 3.2), since this engine already runs real
Postgres via PGlite and already has the `node-db` plumbing
(`server/runner/envs/postgres.mjs`).
*Done when:* the design answers — what is the subject code, what do the
mutants look like for a schema, what budget does a multi-statement transaction
test need (the current 25 s ceiling at `session.ts:305` is your constraint),
and what would make the grader lie — before any build. The build itself is
optional and large.
