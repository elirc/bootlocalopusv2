# The engine course: running someone else's code and grading it honestly

Central question: **what does it take to run someone else's untrusted code and
grade it honestly — without the grader lying, hanging, or being escaped?**

Every lesson in this repository is graded by executing learner code against
real tooling in a worker thread. That makes the *engine itself* the most
senior learning object in the repo: grading untrusted code is a real systems
problem — isolation, resource and time budgets, verdict integrity, flaky-grader
honesty, and progress-state durability. The curriculum teaches the stack; the
engine teaches what it costs to *judge* code written on that stack.

This folder is the learning layer around the engine, in the same discipline as
the three-rebuild course (predict first; done-when anchored to real commands;
evidence at its actual strength; proposed work labelled as exercises):

- [CODE-TOUR.md](CODE-TOUR.md) — one promise traced end to end through the
  grader, submit to XP, ending honestly where the promise stops.
- [EXERCISES.md](EXERCISES.md) — break-and-observe, extend, and
  boundary-moving exercises against the engine.
- [CAPSTONE-TRACK.md](CAPSTONE-TRACK.md) — how the external three-rebuild
  course (GitJira → Stockade → Relay) and its two electives slot into this
  curriculum as the mid-to-senior capstone track.

## Sequence

1. **Stack foundations: the lessons themselves.** The app in this repository
   (`npm start`, then http://127.0.0.1:4517) is the junior→mid programme: 508
   lessons across nine tracks. You do not need all of them before the rest of
   this path, but the engine work below leans hardest on the **Node & API
   Engineering**, **Postgres & Data Modelling** and **Testing & Quality**
   tracks.
2. **The existing mid→senior programme.** [../README.md](../README.md) and
   [../ROADMAP.md](../ROADMAP.md): a competency model and ~22 missions that
   use this codebase as the vehicle, plus the adversarial review pairs in
   [../reviews/](../reviews/) and the journal in [../journal/](../journal/).
   Read the roadmap end to end once. The review pairs are the course's best
   single artifact: review each area yourself *before* reading the pair, then
   calibrate.
3. **Trace the grading promise.** Take [CODE-TOUR.md](CODE-TOUR.md) with the
   files open. Before you start, read
   [../journal/005-where-the-trust-boundary-actually-is.md](../journal/005-where-the-trust-boundary-actually-is.md) —
   it records the three one-liners that forged a passing grade in v1 and the
   argument about what "sandbox" should even mean here.
4. **Break it on purpose.** [EXERCISES.md](EXERCISES.md) Tier 1, then the
   Tier 2 extensions (author a real lesson end to end; prove grader
   determinism; corrupt the save file and watch recovery).
5. **The capstone track.** [CAPSTONE-TRACK.md](CAPSTONE-TRACK.md): the three
   rebuilds and the two electives, with this curriculum's chapters as named
   prerequisites. Their own verification commands are the grading.
6. **Move a boundary.** One Tier 3 exercise, one page of design first.

## Ground rules (inherited from both programmes)

- Predict before you run. The learning is in the gap between prediction and
  output.
- Evidence at its actual strength: "the smoke suite passes" and "the sandbox
  cannot be escaped" are different claims; say which one you have. (The second
  one is false here, on purpose — see the end of the code tour.)
- Never claim output you did not capture. Every quoted result in these
  documents comes from a run that actually happened; keep yours to the same
  standard.
- Exercises are labelled exercises. Nothing in this folder pretends to be
  implemented when it is not.
- Keep `npm test` green between exercises, and never commit a deliberate
  break, a `data/progress.json` change, or anything under `.runs/`.
