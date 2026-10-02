# Code tour: one promise, submit to XP

This tour follows a single promise through every layer of the engine:

> **A submit earns XP only when the learner's code really passed the lesson's
> tests — the verdict cannot be forged from inside the sandbox, a grader
> failure is never scored against the learner, and a pass, once paid, is never
> silently lost or repriced.**

Read it with the files open. Line numbers refer to the current source; if they
drift, search for the quoted code instead. The example lesson is
`node-http-router` (kind `node`), because it shows the grader at its most
real: the learner's code opens an actual HTTP server and the spec hits it with
actual `fetch`.

Before each section, predict what the layer must do — then check.

## 1. The submit

- `web/src/api.ts:269` — the client POSTs `{ code }` to
  `/api/lesson/:id/submit`. Nothing else travels; the client never computes or
  sends a verdict.
- `server/app.ts:61` — before any route: the Host header must be loopback
  (`localhost` / `127.0.0.1` / `[::1]`), because this server runs arbitrary
  code and must never be reachable from another machine's browser. `:65` caps
  the JSON body at 128 KB; `:33` caps code at 60,000 characters.
- `server/app.ts:230` — the submit handler. For non-quiz kinds it saves the
  draft *first* (`:239` — a reload never loses work), then grades.
- `server/app.ts:102-115` — the grading queue: one grading runs at a time in
  the whole process (`queue`), and a second submit of the *same* lesson while
  one is in flight is a 409. Predict: why serialize all gradings rather than
  just per-lesson? (Hint: PGlite, jsdom and `tsc` are heavy, and the budget is
  wall-clock — a loaded machine would make honest code time out.)

## 2. The worker-thread boundary — what is actually isolated

- `server/runner/index.ts:57` — `runExercise` picks the budget
  (`session.ts:305`: 10 s for `js`/`node`, 20 s for `react`/`typecheck`, 25 s
  for `sql`/`node-db` — armed only once the environment is ready) and hands
  off to `runSession`.
- `server/runner/session.ts:190-201` — the spawn. Read the options; each one
  is a lesson learned the hard way (the v1 bugs behind them are in
  `upskilling/journal/005-*.md`):
  - a **private `MessageChannel`** is transferred in; the ordinary
    `parentPort` is never read (`:282` — `worker.on('message', () => {})`).
    Learner code can reach `parentPort`; it cannot reach `port1`.
  - `execArgv: []` — no inherited `tsx` loader, so learner code cannot import
    the server's TypeScript (in v1 it could read every reference solution).
  - `env` is replaced wholesale (`NODE_ENV: 'sandbox'` plus a per-run TMPDIR):
    no inherited environment.
  - `resourceLimits: { maxOldGenerationSizeMb: 512 }` — a heap ceiling, and
    only the JS old generation; this is a guard rail, not a quota.
- `server/runner/worker.mjs:33-49` — inside the worker, the job descriptor and
  the private port are copied into a closure and `workerData` is scrubbed, so
  learner code that imports `node:worker_threads` finds nothing useful there.
- `server/runner/sandbox/guard.mjs` — the lockdown, installed *after* the
  environment boots and *before* learner code loads:
  - `:29-40` — `process.exit`, `kill`, `abort`, `chdir`, `dlopen`, `binding`,
    `getBuiltinModule` … are replaced with throwing stubs (in v1,
    `process.kill(process.pid)` took the whole server down — workers share
    the PID).
  - `:42-44` — imports of `child_process`, `worker_threads`, `vm`, `module`,
    `v8`, `inspector` are denied by a resolve hook (`:54-75`), as is any
    `file:` import outside the run dir and the project's `node_modules`.
  - `defineLocked` (`:78`) makes `describe`/`it`/`expect` and friends
    non-writable, non-configurable globals — the v1 forgery
    `globalThis.expect = () => …` now throws.
  - `server/runner/sandbox/primordials.mjs` — the trusted core calls captured
    builtins, so `Array.prototype.every = () => true` (v1 forgery #3) cannot
    bend the harness.

**Now read `guard.mjs:1-10` closely, because it is the most honest comment in
the repo:** *"None of this is a security boundary. The code still runs in this
thread, with `fs` and `net` (the node track needs them)."* Hold that thought
for §7.

## 3. Boot is not the learner's time

- `worker.mjs:5-11` — the order is the design: environment setup (jsdom,
  PGlite, the `tsc` program) happens *before* `ready` is posted.
- `session.ts:229` — until `ready`, the only clock is the 60 s boot budget;
  `:236-240` — at `ready`, the learner's budget is armed. A slow machine
  loading PGlite cannot eat the learner's 10 seconds.
- `server/runner/index.ts:28-31` — `ready: false` on a failure means "the
  environment's fault", and `server/app.ts:121-122` (`isScored`) refuses to
  count such a run as an attempt. This is **flaky-grader honesty encoded as a
  predicate**: a boot timeout, a crashed worker, or a lesson whose grader
  registered no tests (`index.ts:95-98`) costs the learner nothing.

## 4. The grading itself (kind `node`)

- `worker.mjs:185-228` (`runGraded`) — the learner's file is transpiled and
  imported; its exports become the locked `solution` global (`:180`); then the
  lesson's spec is imported and the harness runs.
- `content/node/http/node-http-router/tests.js:1-10` — the spec calls
  `solution.createServer()`, `listen(0, '127.0.0.1')` — a **real socket on a
  real ephemeral port** — and grades with real `fetch` (`:16-19`: status,
  `content-type`, JSON body). No string comparison against expected source
  anywhere; a router that mishandles `?query` strings fails test `:23` on
  observable behavior.
- Each test row is streamed to the parent as it finishes (`worker.mjs:144-147`
  — `onTest`), so a later hang cannot retroactively erase earlier results.

## 5. The verdict is computed by the parent, from rows

- `session.ts:120-137` — every field of every message is normalised to its
  declared type and length-capped; `session.ts:2-8` states the rule: *the
  worker ran learner code, so every message it sends is untrusted data*.
- `server/runner/index.ts:100` — the only line that decides pass/fail:

  ```ts
  const ok = !error && tests.length > 0 && tests.every((t) => t.passed === true);
  ```

  Nothing the worker says about `ok` is ever read (there is no `ok` in the
  protocol at all — see `worker.mjs:17-21`). The v1 forgery of posting
  `{ ok: true }` on `parentPort` now lands in a listener that discards it.
- Timeouts stay honest: `index.ts:78-87` — on a budget timeout, the rows that
  finished are kept, the one in flight is marked `timed out`, the rest `not
  run`. The learner sees *which* test hung.

## 6. Paying the pass, and keeping it paid

- `server/app.ts:246-250` — the comment is the contract: everything after the
  `await` is synchronous, and the record is re-read *after* the await, so a
  parallel submit of the same lesson cannot pay twice (belt on top of the
  `inFlight` 409 suspenders).
- `server/rewards.ts:67-87` (`applyPass`) — XP is computed from hints used and
  reveals, then `:87` freezes a `pass` snapshot: *"hints opened after passing
  must never change what this was worth."* The economy's honesty rule, in one
  line.
- `server/progress.ts:223-244` (`writeAtomic`) — write to a temp file,
  `fsync`, then rename over `data/progress.json`, with a retry loop for
  Windows' rename-while-read refusal. `:274-294` — saves are serialised and
  coalesced. `:246` — a daily backup before the first write of a day; `:202`
  (`quarantineAndRecover`) — an unreadable save is moved aside, never
  overwritten, and the newest backup restored. `:308` — a lock file refuses a
  second server on the same data dir.

## 7. Where the promise stops

Report of what is actually in the code, not what "sandbox" usually implies:

- **The sandbox is a worker thread, not a security boundary — and the code
  says so** (`guard.mjs:6-9`). Learner code runs with full `node:fs` and
  `node:net` access as the *same OS user*, because `node` lessons genuinely
  need sockets and files. The import policy stops `import`/`require` of files
  outside the run dir; it does not stop `fs.readFile` of any path on the
  machine — including `data/progress.json` itself, or your SSH keys. The
  threat model (`journal/005-*.md`) is explicit: the learner and the operator
  are the same person on their own laptop; the guards exist to stop
  *accidents* and *grade forgery*, not malice. Running this engine for code
  you did not write yourself would need a process- or VM-level boundary —
  that is Tier 3 exercise 3.1, not a property it has today.
- **Network is open.** Graded code can `fetch` the internet. The app makes "no
  network call at runtime" (README) — learner code is not so bound.
- **CPU and memory limits are coarse.** The budget is a parent-side wall-clock
  timer answered by `worker.terminate()`; there is no CPU quota, and
  `maxOldGenerationSizeMb` does not cover `Buffer` allocations (a v1 review
  finding; the limit set at `session.ts:198` is the heap only).
- **Determinism is engineered, not guaranteed.** Serialised gradings, boot
  excluded from the budget, per-test timeouts — but the spec still runs on a
  shared machine's clock. Whether a verdict is identical across 20 runs is an
  empirical question, and Tier 2 exercise 2.2 makes you measure it rather
  than assume it.

When you can explain why the verdict line in `index.ts:100` plus the private
port is sufficient against a *grade-forging* learner, and why nothing short of
a separate process (or VM) would be sufficient against a *hostile* one, you
have the lesson this engine teaches.
