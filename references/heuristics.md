# Hunting Heuristics — Menu by Surface

Use this as a menu, not a checklist. Each wave, prioritize surfaces matching the `--theme` (if given) or the codebase's highest-risk categories.

## Authentication / Session

- Login returns distinct error strings for "email not found" vs "wrong password" → email enumeration
- Password reset token: entropy, single-use, expiry, bound to user vs bearer
- Session / logout: does logout revoke server-side refresh tokens, or only clear cookies?
- JWT claims staleness: does role revocation take effect before token expiry?
- Session fixation: does login rotate the session token?
- Rate limiting on auth endpoints: missing? in-memory (defeated by cold starts)?
- HMAC/signature comparison: is it timing-safe?

## Authorization / RLS / Multi-tenancy

- RLS policies with `USING (true)` on tenant tables
- `service_*` policies with `WITH CHECK (true)` and no `TO service_role` clause
- `SECURITY DEFINER` functions without `REVOKE EXECUTE FROM anon, authenticated, public`
- UPDATE policies that check row ownership but not column scope (admin columns inherit public-read)
- Cross-tenant queries missing tenant_id filter
- JWT claims trusted without DB re-verification where it matters
- Staff/role invitations: email-match check missing, role upgrade via self-UPDATE

## Schema / Migrations

- `ADD COLUMN` on a table with permissive RLS that should have been narrowed
- Missing `CHECK` constraint on enum-like columns
- Missing `UNIQUE` / `NOT NULL` on important columns
- Non-idempotent migrations that claim to be idempotent
- Missing indexes on FKs used in RLS predicates
- RPC functions called but never defined
- Column named one thing, queried as another (runtime failures hidden by mocks)

## Input Validation

- Zod / schema validation missing at route boundaries
- Array length not capped on batch operations (check the RPC accepting `ticket_data: T[]`)
- File upload: MIME whitelist missing, size limit missing, path traversal in object key
- SVG uploads → stored XSS
- URL/redirect parameters not validated as same-origin (open redirect)
- Command injection (shell_exec, child_process with user input)
- SQL/NoSQL injection in dynamic query construction
- Prototype pollution in Node.js
- Server-side template injection

## Concurrency / TOCTOU

- Reservation → confirm flow: atomic? locked?
- Check-in: `SELECT ... then UPDATE` without transaction
- Inventory / sold_count: row locking, `SKIP LOCKED`, `FOR UPDATE`
- Unique constraint races where the app does "check then insert"

## Economic Logic

- Oversell possible: stock check not atomic with reservation
- Undercharge: total computed from reservation but line items from client
- Free upgrade: tier_id validation missing
- Discount stacking: multiple codes applied
- Negative quantities, decimal precision on money
- Refund flows: authorization? idempotency?
- Payment webhook: signature verification, replay

## Side Channels

- Error-message differentials (found vs not-found, 401 vs 403)
- Timing attacks on token compare
- UNIQUE-constraint 23505 surfacing to client → enumeration
- Log files / stack traces in error responses
- Response size differences for authorized vs unauthorized

## Build / CI / Supply Chain

- `continue-on-error: true` on critical steps
- `|| echo` masking non-zero exits
- Tests that pass with mocked I/O but would fail with real I/O
- `@latest` or loose caret ranges in security-critical deps
- postinstall scripts in dependencies
- Lockfile drift (manual edits, mismatched versions)
- Missing CI gates (migrate, lint, typecheck, coverage minimum)

## External Boundaries

- Webhook signature: verified? Timing-safe compare? Replay prevention via idempotency key?
- CORS: wildcard origin on cookie-auth endpoints
- CSP: missing, or `unsafe-inline` / `unsafe-eval`
- HSTS, X-Frame-Options, Referrer-Policy, COOP
- Image proxy / Next Image `remotePatterns` — SSRF?
- Server-side fetch of user-supplied URLs (SSRF to internal services / AWS IMDS)

## Client-Side Persistence

- Tokens in localStorage (any)
- `persist` middleware (Zustand, Redux persist) — what's stored?
- sessionStorage secrets
- IndexedDB
- Cookie attributes: Secure, HttpOnly, SameSite, Domain scope

## Logs / Audit Integrity

- Audit log INSERT policy trusts client-supplied `actor_id`
- Audit log is UPDATE-able or DELETE-able
- Actions logged inconsistently (some paths skip logging)
- Logs include secrets, PII, tokens

## Realtime / Pub-Sub

- Channel authorization on subscribe
- Broadcast payloads leaking across tenants
- Presence data exposure

## i18n / Localization

- Key parity between locale files
- Hardcoded strings in production code
- Interpolation injection (user data into translation keys)
- RTL / Unicode handling

## Error Boundaries / Information Disclosure

- Error pages rendering `error.message`, `error.stack`, SQL errors, or column names in production
- Debug endpoints left enabled (`/api/debug`, `/api/dev`, `/health?verbose=true`)
- `.env` or secrets files tracked in git
- Sourcemaps served in production

## Framework-Specific Traps

### Next.js
- `dynamic = 'force-dynamic'` missing where caching would leak
- Server Actions CSRF (built-in, but custom route handlers don't get it)
- Async params/searchParams not awaited (Next 15+)
- Middleware bypass via `skipMiddlewareUrlNormalize` or matcher gaps
- Image proxy SSRF
- `revalidatePath` / cache invalidation missing, leaking stale data

### Supabase
- Mixing `createClient()` (browser) with server code or vice versa
- Using `@supabase/auth-helpers-nextjs` instead of `@supabase/ssr` for App Router
- Service role key exposed in `NEXT_PUBLIC_*`
- `@yudiel/react-qr-scanner` server-rendered (known-broken)
- `auth.admin.*` called with anon key (silent failure)
- RLS tested via SQL Editor (bypasses RLS)
- Realtime channels without authz policy on `realtime.messages`

### React / TypeScript
- `dangerouslySetInnerHTML` with unsanitized user input
- `href={userUrl}` without scheme validation (javascript:)
- `target="_blank"` without `rel="noopener"`
- Zustand `persist` on state containing sensitive data

## Meta-Heuristics

- **Look for the gap between commit waves.** If commits show "added X column" and a later commit shows "added Y policy for X column," grep to ensure every `ADD COLUMN` got a policy audit.
- **Check the comment-vs-code mismatch.** Comments saying "service role only" on a policy with no `TO service_role` clause — 9 times out of 10, bug.
- **Read tests for the bug.** If the test mocks the risky boundary, the boundary is untested.
- **Trace from public entry.** Start from a publicly-reachable URL/endpoint and walk inward. If no auth check appears by the time you hit a sensitive DB write, something's wrong.
