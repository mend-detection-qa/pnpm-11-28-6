# optional-peer-resolution-v11

Probe targeting pnpm 11.28.6 optional peer dependency
resolution changes.

## Feature exercised

pnpm 11.28.6 introduced changes to optional peer dependency
resolution logic that affect how the Unified Agent (UA)
parses and resolves dependency trees — specifically for
packages that declare optional peers (via
`peerDependenciesMeta.<name>.optional: true`).

This probe exercises the two distinct resolution contexts for
an optional peer within a single lockfile:

1. **Satisfied optional peer** — `ajv@8.17.1` is installed at
   the root alongside `typescript@5.4.5`. Because `typescript`
   is present, pnpm records the snapshot as
   `ajv@8.17.1(typescript@5.4.5)`, with `typescript` as a
   concrete dependency in the snapshot. Mend must include
   `typescript` in `ajv`'s dependency list.

2. **Absent optional peer** — `hono@4.4.13` is installed
   without any optional peers. Its snapshot has no conditional
   suffix. Mend must NOT synthesize a peer dependency for
   `hono` based on the `ajv` snapshot context.

The probe intentionally does NOT use two workspace packages —
the root `package.json` is a flat single-project setup. The
two resolution contexts are encoded directly in the lockfile:
one snapshot for `ajv` that carries the optional peer, and
one that does not (for `hono`, which has no optional peers).

## Dependency graph

```
optional-peer-resolution-v11@1.0.0
  ├── ajv@8.17.1            (registry, snapshot: with typescript)
  │   ├── fast-deep-equal@3.1.3    (registry)
  │   ├── json-schema-traverse@1.0.0  (registry)
  │   ├── require-from-string@2.0.2  (registry)
  │   └── typescript@5.4.5  (optional peer, satisfied, optional: true)
  ├── hono@4.4.13           (registry, no optional peers)
  └── typescript@5.4.5      (registry, satisfies ajv's optional peer)
```

## Expected dependency tree

Key expectations:

- `ajv@8.17.1` is a direct dependency of root.
- `typescript@5.4.5` is BOTH a direct dep of root AND listed
  as a child of `ajv` with `optional: true` (it was resolved
  as ajv's optional peer in the `(typescript@5.4.5)` snapshot
  context).
- `hono@4.4.13` is a direct dep of root with no children
  (its snapshot carries no transitive deps).
- `fast-deep-equal`, `json-schema-traverse`,
  `require-from-string` are transitive children of `ajv`.
- The `optional: true` flag MUST be set on the `typescript`
  entry inside `ajv`'s dependency children context.
- `typescript` at the root level has `optional: false`
  (it is a direct, non-optional dep of the project itself).

## Mend failure modes targeted

| Failure mode | Expected result if bug present |
|---|---|
| Optional peer included for non-peer context | `hono` would spuriously list `typescript` as child |
| Optional peer dropped for satisfied context | `ajv`'s children would be missing `typescript` |
| `optional: true` flag missing | `typescript` under `ajv` shown as hard dep |
| Duplicate peer entries | `typescript` appears twice under `ajv` |
| Snapshot suffix `(typescript@5.4.5)` not parsed | `ajv` dependencies missing; UA may fail to parse snapshot |

## pnpm version

`pm_version_under_test: "11.28.6"` — this probe was built
against the pnpm 11.28.6 optional peer resolution change
(category: tree_structure). The lockfile is v9 format.

## Mend config

**Bucket A** — `js-pnpm` has no dynamic version detection
from the manifest. The `.whitesource` file pins:

- `pnpm: "11.28.6"` — exact version of the change under test
- `node: "20.11.1"` — LTS node, pinned for reproducibility

`configMode` is `"AUTO"` (no `whitesource.config` ships with
this probe). No additional `.whitesource` blocks are needed —
this probe does not target branch scoping, project-token
routing, or suppression gating.

## Files

| File | Purpose |
|---|---|
| `package.json` | Root manifest with direct deps |
| `pnpm-lock.yaml` | v9 lockfile encoding both peer contexts |
| `.whitesource` | Bucket-A versioning pin for reproducibility |
| `expected-tree.json` | Ground truth for downstream comparison |
| `README.md` | This file (probe metadata) |
