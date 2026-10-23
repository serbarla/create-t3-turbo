# Unresolved divergence — `@acme/api` typecheck/lint red

- Source spec: `specs/02-B-database-drizzle-to-mikroorm-kysely.md` → divergence #1, testing criteria (`pnpm typecheck`, `pnpm lint`).
- Status: **OPEN — deferred to spec 04 by design, not a defect.**
- Recorded: 2026-10-23.

## What is broken

`pnpm typecheck` and `pnpm lint` (workspace-wide `nx run-many`) fail **only** in `@acme/api`. Every other target is green (`@acme/db`, `@acme/auth`, `@acme/validators`, `@acme/ui`, tooling).

`@acme/api` still imports Drizzle-era exports that Phase B deleted from `@acme/db`:

- `packages/api/src/router/post.ts` → `import { desc, eq } from "@acme/db";` and `import { CreatePostSchema, Post } from "@acme/db/schema";`
- `packages/api/src/trpc.ts` → `import { db } from "@acme/db/client";`

Secondary (masked by the above): `packages/api`'s `tsconfig` lacks `@types/node`, so `Headers`, `setTimeout`, `console`, `node:sqlite`, `process` also error — but fixing these does **not** clear the 4 primary import errors.

## Why it was not fixed (3 attempts, all fenced)

The cause is the `@acme/db` rewrite itself; fixing it requires work that Spec 02 Section 2 (No-go) explicitly forbids:

1. **Rewrite `@acme/api` off Drizzle** → No-go: "No `@acme/api` / route work (spec 04)". The real fix is the tRPC → Elysia + Eden rewrite owned by spec 04, a much larger scope.
2. **Add compat shims** re-exporting `desc`/`eq`/`Post`/`CreatePostSchema` and `@acme/db/client`/`./schema` → No-go: "Do not keep `drizzle-orm` … as transitive re-exports, compat shims."
3. **Patch `packages/api/tsconfig.json`** (add `@types/node`) → clears only the secondary errors; the 4 import errors remain, and editing `@acme/api` is still out of scope.

Conclusion: the red is an intentional consequence of the phased cutover. Phase B deletes the old DB surface; the consumer (`@acme/api`) is migrated in a later phase. No shim was added, per the no-go.

## How to clear it (future check-up)

- [ ] Run **spec 04** (`specs/04-D-api-trpc-to-elysia-eden.md`) — rewrite `@acme/api` to Elysia + Eden and stop importing `@acme/db` Drizzle exports.
- [ ] After spec 04, re-run from `create-t3-turbo/`:
  - `pnpm typecheck` → expect exit 0
  - `pnpm lint` → expect exit 0
- [ ] When both pass, mark the two `- [ ]` criteria in `specs/02-B-database-drizzle-to-mikroorm-kysely.md` as `[x]`, and this file can be archived/removed.

## Verification snapshot (2026-10-23)

- `pnpm why drizzle-orm` / `pnpm why drizzle-kit` → empty; no drizzle in tracked source (only stale `node_modules/.bin` shims + `.cache`).
- `@acme/db`: `typecheck`, `build`, `lint`, `format`, migration, WAL, Kysely smoke — all green.
- `@acme/auth`: `typecheck` + `lint` green (migrated to better-auth in spec 03).
