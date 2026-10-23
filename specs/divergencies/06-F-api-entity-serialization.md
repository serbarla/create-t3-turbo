# 06-F — API post routes return `[object Object]` (MikroORM entity serialization)

- Source spec: `specs/06-F-expo-trpc-to-eden.md`.
- Blocked criteria (spec 06, section 3):
  - "List shows posts created from web or curl; **opening a post renders its
    title/content**; creating a post appears without a relay; deleting removes
    it."
  - objective 7 ("prove the Expo client end to end") is partially blocked.
- Owner of the fix: **`packages/api/src/router/post.ts`** (spec 04's file).
  No later spec owns it; this needs a follow-up fix or a reopen of spec 04.

## What is broken

`packages/api/src/router/post.ts` returns MikroORM entity instances from Elysia
handlers. Elysia's response mapping stringifies them, so the HTTP body is the
literal text `[object Object]`:

```
GET    /api/posts       -> 200 [{"id":...}]            (Kysely rows, unaffected)
GET    /api/posts/:id   -> 200 [object Object]
POST   /api/posts       -> 200 [object Object]
DELETE /api/posts/:id   -> 200 [object Object]
```

`GET /api/posts` is fine because it goes through `orm.em.getKysely()` and
returns plain objects. The other three use `em.findOne` / `em.create` /
`em.remove` and return the entity directly.

Impact: `apps/expo/src/app/post/[id].tsx` gets a string instead of a post, so
the detail screen renders empty title/content (the `Stack.Screen` title is
`undefined`). The Expo list/create/delete flows still work because they refetch
the list and never read the mutation response body. Web's create/list also
refetch, so spec 05 did not catch it.

## Why each fix attempt was fenced

- Editing `packages/api/src/router/post.ts` in spec 06 is forbidden — spec 06
  section 2 No-go: "Do not modify `apps/web`, `apps/server`, `packages/*`
  sources, Docker, or CI."
- A client-side workaround (fetching the list and filtering by id) would mask
  the API defect and contradict objective 5's mandated by-id Eden call.
- No fix was attempted; the probe only documented the response.

## How to clear it

1. Serialize entity responses as plain objects in
   `packages/api/src/router/post.ts`:
   - `GET /api/posts/:id`: `const post = await em.findOne(Post, { id }); return post ? post.toObject() : null;`
     (or `wrap(post).toObject()` from `@mikro-orm/core` if the entity lacks
     `toObject`).
   - `POST /api/posts`: same conversion before returning.
   - `DELETE /api/posts/:id`: convert the removed post (or return `{ id }`).
2. Re-verify:
   ```
   pnpm nx run @acme/api:build
   curl -s http://localhost:3000/api/posts | head -c 200        # JSON array
   # with a session cookie:
   curl -s -H "Cookie: better-auth.session_token=..." \
     http://localhost:3000/api/posts/<id>                       # JSON object, not [object Object]
   pnpm typecheck && pnpm lint
   ```
3. Re-run the Expo detail screen (or the HTTP probe) and confirm title/content
   render.
4. When fixed: mark the matching criteria `[x]` in
   `specs/06-F-expo-trpc-to-eden.md` and delete this file.
