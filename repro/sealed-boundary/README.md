# Repro: `Wasm: sealed boundary adaptation missing` — third shape

Self-contained three-module crate extracted verbatim from our query corpus:
`draft` (2119 lines) + `draft_issue` (137) + `draft_alias` (64). No external
dependencies beyond the standard library.

```
cd repro/sealed-boundary
telora lock
telora check @src/draft
```

Expected on main `7e1709a1` (verified on Windows, rustc 1.98.1, debug build):

```
Wasm: sealed boundary adaptation missing: TypeId(631) -> TypeId(373) at HirId(9770)
```

This is the module's only diagnostic (exit code non-zero, no other errors).

## Trigger shape (differs from both shapes fixed in #198)

`draft.decode_hop` — a closure **capturing** an enum value delivered through
`Ok(Forward)` (member-imported unit constructor of the local `DraftDirection`)
plus an Ok-pattern match binding, placing it into an unanchored record literal,
passed to the generic `result.map`, at the declared nominal return boundary
`Result(HopDto, DraftIssue)`:

```telora
Ok(target_raw) => result.map(decode_node(target_raw), fn(target) {
    { alias: alias, relation_ref: relation_ref, direction: direction, target: target }
}),
```

## Why the repro is module-scale (bisection evidence)

| Included content | Result |
|---|---|
| decode_hop's full downward closure (22 items, verbatim) | clean |
| + all sibling decode_* functions (~30, same map/closure/record pattern) | clean |
| + dto helper family | clean |
| + normalize/core/check families (= full module content, export only `decode_hop`) | **triggers** (same TypeId/HirId) |

The failure requires the decode→normalize pipeline **co-resident in one
module**; no line-level minimal repro exists for this bug. Four hand-written
~40-line variants of the same shape all pass.

## Why we believe the source is valid

1. The diagnostic is emitted **after** the solver accepted and sealed the MIR
   — it is not a type error. A compiler must either reject with a located
   diagnostic or emit; an unlocatable post-acceptance emitter failure is a
   compiler defect by contract.
2. The same modules check clean on the previous engine generation (Sep 12).
3. The same engine accepts the same pattern 14 times in our revived copy
   (including an enum field, but delivered as the map parameter rather than a
   captured binding).
4. `adapt()` still has the original three conversion classes; both #198 fixes
   added evidence-recording for specific shapes without expanding the matrix.

## Related observations

- Rewriting `result.map(f, closure)` as an equivalent nested `match` makes the
  engine hang (300s+ no response) — suspected separate hang bug.
- This diagnostic carries no source labels (all occurrences we have seen).
- Batch `check --lib` systematically under-reports this error class: with
  static errors present anywhere, the batch session skips the execution phase;
  per-module isolated checks are required to observe it.
