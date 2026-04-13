# AMKB Conformance

This directory holds the **human-readable conformance test matrix**
for AMKB. It is the authoritative list of behaviors an implementation
must exhibit to claim a given conformance level.

The matrix is maintained as Markdown test lists, one file per level.
Each test carries an ID, a one-sentence description, a reference to
the normative spec section it verifies, and the expected outcome.

## What is here, and what is not

- **Here**: the list of tests, their identifiers, their intended
  checks, and their mapping to normative sections in `spec/`.
- **Not here**: runnable test code. Executable contract tests belong
  in an SDK repository. The Python reference contract lives in
  [amkb-sdk](https://github.com/takyone/amkb-sdk) as
  `amkb_conformance.*`. Other languages (TypeScript, Rust, Go)
  SHOULD maintain their own contract test suites, each
  authoritatively derived from this matrix.

Keeping the matrix documents-only preserves this repository's scope
(specification, not implementation) and lets multiple SDKs coexist
without privileging any one language.

## Conformance Levels

| Level | Name          | File                              |
|-------|---------------|-----------------------------------|
| L1    | Core          | [L1-core.md](L1-core.md)          |
| L2    | Lineage       | [L2-lineage.md](L2-lineage.md)    |
| L3    | Transactional | [L3-transactional.md](L3-transactional.md) |
| L4a   | Structural    | [L4a-structural.md](L4a-structural.md)     |
| L4b   | Intent        | [L4b-intent.md](L4b-intent.md)    |
| L5    | Policy        | (forthcoming, optional)           |

## Claiming a Level

An implementation claims a conformance level by stating the level in
its documentation and passing every test listed in the corresponding
file *and* every test in all lower levels. Cherry-picking is not
allowed: an implementation claiming L2 MUST pass all L1 tests and
all L2 tests.

Implementations SHOULD publish a self-assessment pointing at a
specific commit of this matrix (by git SHA). Implementations MAY
claim partial conformance ("L1 core + selected L2 tests") in
documentation but MUST NOT advertise that as a formal level.

## Test IDs

Test identifiers follow the pattern:

```
L<level>.<area>.<seq>
```

where:

- `<level>` is `1`, `2`, `3`, `4a`, `4b`, or `5`.
- `<area>` is a short area name (`create`, `retire`, `merge`,
  `retrieve`, `events`, ...).
- `<seq>` is a two-digit sequence within the area.

Example: `L1.create.03` is the third test in the "create" area of
L1. IDs are stable across spec versions within a major version;
when a test is retired, its ID is not reused.

## Test format (per-test)

Each test in the per-level files is presented as:

```markdown
### L<level>.<area>.<seq> — <title>

**What.** One sentence describing the behavior under test.

**Spec.** A pointer to the normative section(s) that define the
expected behavior.

**Setup.** Preconditions the test establishes before invoking
the operation.

**Action.** The operation or sequence invoked.

**Expected.** The outcome the implementation MUST produce.
```

Tests SHOULD be written at a level of abstraction that any
implementation can execute, independent of SDK, wire format, or
storage backend.

## Status

This matrix is in **draft** while the spec itself is in pre-draft.
Test lists are incomplete: they enumerate the intended coverage
but are not yet exhaustive. Contributions, especially from
implementers, are welcome through the usual repository channels.
