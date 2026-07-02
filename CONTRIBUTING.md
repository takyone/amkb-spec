# Contributing

Issues and pull requests are welcome, especially from implementers
exercising the spec against real code.

- **Normative changes** (anything in `spec/00-overview.md` through
  `spec/05-errors.md` that adds, removes, or alters a MUST/SHOULD/MAY
  rule) MUST be accompanied by a new entry in `spec/99-rationale.md`
  recording the decision, the alternatives considered, and the
  reasons for the choice. PRs that change normative text without a
  rationale entry will be asked to add one before merge.
- **Editorial changes** (typos, cross-reference fixes, clarifying
  prose that does not change behavior) do not need a rationale entry
  but should say so in the PR description.
- **Conformance matrix changes** (`conformance/*.md`) should point at
  the specific spec section(s) they verify and follow the test
  format documented in `conformance/README.md`.
- This specification uses RFC 2119 / RFC 8174 keywords normatively.
  Keep that discipline: MUST/MUST NOT/SHOULD/SHOULD NOT/MAY/OPTIONAL
  in all capitals are load-bearing; do not introduce ad hoc synonyms
  (e.g. "MAY NOT") that aren't part of the RFC 2119 vocabulary.

See [README.md](README.md) for the reading order and
[spec/99-rationale.md](spec/99-rationale.md) for prior design
decisions before proposing a change that might already have been
considered and rejected.
