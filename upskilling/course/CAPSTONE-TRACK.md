# The capstone track: the three-rebuild course and its electives

The junior→mid curriculum in this repository grades you lesson by lesson. The
mid→senior programme in [../ROADMAP.md](../ROADMAP.md) grades you mission by
mission against this engine. The capstone track sits between and after them:
three small systems built and reviewed adversarially, each asking one sharp
question, each proving its answer with tests you run, read and break — plus
two electives that point the same discipline at dimensions the rebuilds
exclude.

The rebuilds live in their own repositories, each under its `rebuild/` folder:

| # | Repo | Central question | Start at |
| --- | --- | --- | --- |
| 1 | [elirc/gitjira](https://github.com/elirc/gitjira) | Can two editors silently lose each other's work? | `rebuild/` → CODE-TOUR.md, then EXERCISES.md |
| 2 | [elirc/stockade](https://github.com/elirc/stockade) | Can two buyers be promised the same physical unit? | `rebuild/` → journal/LEARNING-PATH.md |
| 3 | [elirc/relay](https://github.com/elirc/relay) | Can a workflow recover when the outside world accepted an effect the worker never checkpointed? | `rebuild/` → journal/LEARNING-PATH.md |

**The grading is theirs, not this engine's.** Each rebuild carries its own
verification commands — `npm run check`, `npm test`, `npm run test:browser`
from its folder — and its exercises define done-when against those suites.
Nothing here re-grades them. (Embedding one of their exercises as a graded
lesson *in this engine* is Tier 3 exercise 3.3 in
[EXERCISES.md](EXERCISES.md) — a design exercise, not something that exists.)

Work them in order; each assumes the discipline the previous one taught. The
connecting thread — every piece of async work needs an owner with the right
lifetime (a row version, a checkout key, a lease token) — is the senior
version of ideas this curriculum's chapters introduce lesson-sized.

## Prerequisites, by rebuild, in this curriculum's own chapters

"Prerequisite" means: the rebuild assumes you can already do this at
lesson scale, so finish (or honestly skip past) these chapters first.

### Before GitJira (optimistic concurrency, CRUD done honestly)

- **Node & API Engineering** → *HTTP from Scratch*, *API Engineering*
  (validation, error envelopes), and above all *API Design in Practice* — its
  `node-conditional-updates` lesson ("ETags and If-Match: no more lost
  updates") is GitJira's compare-and-swap in miniature, and
  `node-cursor-pagination` is its exercise 2.3.
- **Postgres & Data Modelling** → *Modelling Data* (constraints, migrations)
  and *Postgres from code*; from *Concurrency & Locking*, at least
  `node-db-optimistic-locking` — the version-match UPDATE that GitJira's whole
  code tour orbits.
- **Testing & Quality** → *Integration Tests* (tests that talk HTTP to a real
  listener — the only style GitJira's suites use).

### Before Stockade (contention on a shared physical resource)

Everything above, plus:

- **Postgres & Data Modelling** → the rest of *Concurrency & Locking*:
  `node-db-lock-parent-row`, `node-db-lock-ordering`,
  `node-db-serialization-retry`, and the `node-db-seat-hold-boss` ("seat
  hold") boss, which is Stockade's oversell question at lesson scale.
- **Node & API Engineering** → *API Design in Practice*'s
  `node-idempotency-keys` ("retries that do not charge twice") — Stockade's
  checkout keys with intent fingerprints are this lesson grown up — and
  *Defending the Input Boundary*.
- **JavaScript You Actually Need** → *Async, Properly* and *Async Patterns
  II* (single-flight, semaphores) if two-process interleavings still feel
  abstract.

### Before Relay (durable workflows across a store boundary)

Everything above, plus:

- **Node & API Engineering** → *Background Work* end to end:
  `node-job-queue-retries`, `node-dead-letter`, `node-job-leases`,
  `node-db-outbox` ("The transactional outbox") and `node-db-job-lock` —
  Relay's lease fencing and effect/checkpoint gap are these five lessons
  forced to coexist; plus *Resilience II* (`node-retry-policy`,
  `node-deadline-propagation`) and *Observability*.
- **Postgres & Data Modelling** → *Concurrency & Locking*'s
  `node-db-skip-locked-queue` and `node-db-advisory-lock`.

### Around the whole track

- **Engineering Craft** → *Working Like a Mid* and *Communication* (review
  anatomy, postmortems) — the rebuilds' review journals assume you can read a
  finding as *input, observed result, violated promise*.
- The adversarial review pairs in [../reviews/](../reviews/) are the local
  rehearsal for the rebuilds' builder/reviewer/verifier protocol.

## The electives

Two more repositories carry companion courses in the same discipline, each on
a dimension the rebuilds deliberately exclude:

- [elirc/lucasrouter](https://github.com/elirc/lucasrouter) (**RouteIQ**,
  start at `upskill/LEARNING-PATH.md`) — heuristic optimization behind a
  swappable API contract: what you can honestly promise about output produced
  by an algorithm you intend to replace. Local prerequisites: **Software
  Design** → *Designing Module APIs*; **Testing & Quality** → *Tests that
  discriminate* and *Test Strategy & Hygiene* (contract tests that pin
  behavior, not implementation — the `mutation` lessons here train exactly
  that muscle).
- [elirc/aral-tagalog-v2-v2](https://github.com/elirc/aral-tagalog-v2-v2)
  (**Aral**, start at `upskilling/course/LEARNING-PATH.md`) — offline-first
  sync in a web+mobile monorepo: progress recorded offline merging into a
  server account without loss or double counting. Local prerequisites: **Web
  Platform** → *Browser APIs* (storage, events) and *HTTP from the Browser*;
  **Node & API Engineering** → `node-idempotency-keys`; GitJira first, since
  the outbox/reducer/sync path is the identity-and-idempotency theme on a new
  boundary.

## Where this engine re-enters

The rebuilds defer "port to Postgres under acceptance tests" (GitJira
EXERCISES.md 3.2) as their largest boundary move — and this repository already
grades real SQL and real Node-against-Postgres (`sql` and `node-db` kinds, via
PGlite). Designing that port as a *graded chapter here* — subject, mutants for
a schema, budgets, failure modes of the grader itself — is Tier 3 exercise 3.3
in [EXERCISES.md](EXERCISES.md). It is the natural meeting point of the two
courses, and it is explicitly not implemented today.

## Suggested overall order

1. This curriculum to at least Mid-Track rank (the prerequisites above green).
2. The engine course: [CODE-TOUR.md](CODE-TOUR.md), Tier 1–2 of
   [EXERCISES.md](EXERCISES.md), alongside the ROADMAP missions you choose.
3. GitJira → Stockade → Relay, each by its own learning path, each verified by
   its own three commands.
4. One elective, or the rebuild course's order-fulfillment capstone design
   (see its CURRICULUM.md), or Tier 3 here. One boundary at a time.
