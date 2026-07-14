# [MLID-364] fix(proxy): read the `__Secure-` session cookie on HTTPS to stop API 401s

## Summary

Hotfix for the Next.js 16 / Auth.js v5 migration (MLID-364). On the staging deploy, **every non-public `/api/*` route returned 401 Unauthorized** even though the user was logged in and the page shell rendered. The root cause is a cookie-name mismatch in `proxy.ts` (the former `middleware.ts`): the auth gate did not tell `getToken` the request was secure, so on HTTPS it looked for the wrong cookie name and rejected every request.

**Branch:** `fix/MLID-364-proxy-secure-cookie` -> `develop`

## Symptom

On `https://app-stage.mylocalinfusion.ai` the console showed 401 on every authenticated call:

- `POST /api/signalr/negotiate` -> 401 (SignalR / realtime down)
- `GET /api/socket`, `GET /api/socket_io?...` -> 401
- `GET /api/feature-flags` -> 401 ("Error fetching feature flags")
- `GET /api/findProgress`, `POST /api/dictionaries/getDictionaries` -> 401

The GitHub Action was green, and local `npm run dev` was clean — the bug only appears on HTTPS.

## Root cause

`apps/web/proxy.ts` gates every non-public `/api/*` route by validating the session with `getToken` from `next-auth/jwt`:

```js
const token = await getToken({ req: request, secret: process.env.NEXTAUTH_SECRET });
if (!token) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

Auth.js v5 stores the session under a **scheme-dependent cookie name**:

- over HTTP: `authjs.session-token`
- over HTTPS: `__Secure-authjs.session-token`

Without `secureCookie: true`, `getToken` looks for the bare `authjs.session-token`. On the HTTPS staging host the real cookie is `__Secure-authjs.session-token` (confirmed in DevTools -> Application -> Cookies), so `getToken` finds nothing, returns `null`, and the proxy 401s every API route.

Why it was invisible until staging:

- **The page still rendered logged in** — the proxy only guards `/api/*`, not pages. Pages authenticate via server-side `auth()`, which reads the `__Secure-` cookie correctly. So the app looked logged in while every API call it made was blocked.
- **Local dev was clean** — localhost is HTTP, so the cookie has no `__Secure-` prefix and `getToken` finds it. The bug cannot reproduce on HTTP.
- **CI was green** — unit tests and the production build cannot exercise a real HTTPS session, so this class of bug is invisible to the pipeline.

## The fix

Derive `secureCookie` from the request scheme and pass it to `getToken`. Behind the reverse proxy the original scheme is carried in `x-forwarded-proto` (the internal `request.nextUrl.protocol` can read as `http`), so read that first and fall back to the request protocol:

```js
const forwardedProto = request.headers.get('x-forwarded-proto')?.split(',')[0]?.trim();
const secureCookie = forwardedProto === 'https' || request.nextUrl.protocol === 'https:';
const token = await getToken({
  req: request,
  secret: process.env.NEXTAUTH_SECRET,
  secureCookie,
});
```

`x-forwarded-proto` is split on `,` because chained proxies can send a list (`https, http`) — the same pattern the file already uses for `x-forwarded-for`. On HTTPS this makes `getToken` read `__Secure-authjs.session-token`; on local HTTP it still reads `authjs.session-token`.

**No new environment variables.** `secret` still uses the existing `NEXTAUTH_SECRET`, and `trustHost: true` is already set in `auth.ts`.

## Files changed

- `apps/web/proxy.ts` — derive and pass `secureCookie`; correct the misleading comment.
- `apps/web/lib/apiAuth.test.ts` — 4 regression tests.

## Tests (TDD)

Added to the existing proxy test suite (`getToken` mocked; assertions on the arguments it receives):

- HTTPS via `x-forwarded-proto: https` -> `secureCookie: true`
- `https://` request URL -> `secureCookie: true`
- comma-separated `x-forwarded-proto: 'https, http'` -> `secureCookie: true`
- plain HTTP request -> `secureCookie: false`

Result: **10/10 pass** in `lib/apiAuth.test.ts`; `tsc --noEmit` clean; `npm run build:web` green.

## Test plan (staging, after deploy)

- [ ] Sign in on `app-stage`, open DevTools console — no 401s on `/api/*`.
- [ ] SignalR / realtime connects (`/api/signalr/negotiate` returns 200; socket connects).
- [ ] Feature flags load (no "Error fetching feature flags").
- [ ] Dictionaries and other authenticated calls succeed.
- [ ] Local `npm run dev:web` still works (HTTP path unaffected).

## Jira

- [MLID-364](https://localinfusion.atlassian.net/browse/MLID-364)
