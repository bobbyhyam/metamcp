# Plan — Implement OAuth Refresh Tokens in metamcp

## Context

claude.ai disconnects from metamcp ~1 hour after initial OAuth (via Authentik → better-auth → metamcp's own OAuth 2.1 server) and demands re-authentication. Investigation traced the cause: the metamcp OAuth server hardcodes `access_token` `expires_in` to 3600 seconds (`apps/backend/src/routers/oauth/token.ts:175`), advertises `refresh_token` grant support in its discovery metadata (`apps/backend/src/routers/oauth/metadata.ts:122`), but never actually issues refresh tokens — and the `/oauth/token` endpoint explicitly rejects every grant_type except `authorization_code` (`token.ts:36-41`). claude.ai's MCP traffic is server-to-server with bearer tokens only, so once the access token expires there is no path to silently refresh; the user gets a "disconnected" prompt.

This plan adds spec-correct OAuth 2.1 refresh-token issuance and rotation so claude.ai (and any other MCP client) can hold a long-lived connection without re-prompting the user every hour.

Out of scope: migrating to better-auth's native OIDC provider plugin (10× the blast radius — separate refactor); reuse-detection enforcement (schema is forward-compatible so v1.1 enforcement is logic-only).

---

## Files to modify

| # | Path | Change |
|---|------|--------|
| 1 | `apps/backend/src/db/schema.ts` | New `oauth_refresh_tokens` table |
| 2 | `apps/backend/src/routers/oauth/utils.ts` | Add `generateSecureRefreshToken()` |
| 3 | `apps/backend/src/db/repositories/oauth.repo.ts` | Add refresh-token CRUD + rotation; extend `cleanupExpired` |
| 4 | `apps/backend/src/routers/oauth/token.ts` | Read TTLs from env; issue refresh_token in `authorization_code` branch; add `refresh_token` grant branch; extract `authenticateClient` helper |
| 5 | `packages/zod-types/src/oauth.zod.ts` | Add `OAuthRefreshTokenSchema` mirroring `OAuthAccessTokenSchema` |
| 6 | `example.env` | Document `OAUTH_ACCESS_TOKEN_TTL_SECONDS` (default 3600) and `OAUTH_REFRESH_TOKEN_TTL_SECONDS` (default 2592000 = 30d) |

`apps/backend/src/routers/oauth/metadata.ts` needs **no change** — line 122 already advertises `refresh_token`; it just becomes truthful.

---

## 1 — Schema (`apps/backend/src/db/schema.ts`)

Append after `oauthAccessTokensTable` (line 486):

```ts
export const oauthRefreshTokensTable = pgTable(
  "oauth_refresh_tokens",
  {
    refresh_token: text("refresh_token").primaryKey(),
    client_id: text("client_id")
      .notNull()
      .references(() => oauthClientsTable.client_id, { onDelete: "cascade" }),
    user_id: text("user_id")
      .notNull()
      .references(() => usersTable.id, { onDelete: "cascade" }),
    scope: text("scope").notNull().default("admin"),
    access_token: text("access_token").references(
      () => oauthAccessTokensTable.access_token,
      { onDelete: "set null" },
    ),
    replaced_by: text("replaced_by"),                  // forward-compat for reuse detection
    revoked_at: timestamp("revoked_at", { withTimezone: true }), // forward-compat
    expires_at: timestamp("expires_at", { withTimezone: true }).notNull(),
    created_at: timestamp("created_at", { withTimezone: true })
      .notNull()
      .defaultNow(),
  },
  (table) => [
    index("oauth_refresh_tokens_client_id_idx").on(table.client_id),
    index("oauth_refresh_tokens_user_id_idx").on(table.user_id),
    index("oauth_refresh_tokens_expires_at_idx").on(table.expires_at),
  ],
);
```

Run `pnpm db:generate` (or `db:generate:dev` against `.env.local`) — drizzle-kit emits `apps/backend/drizzle/0014_<name>.sql`. Review the SQL, then `pnpm db:migrate`.

---

## 2 — Utils (`apps/backend/src/routers/oauth/utils.ts`)

Mirror the existing pattern (next to `generateSecureAccessToken`):

```ts
export function generateSecureRefreshToken(): string {
  return `mcp_refresh_${randomBytes(32).toString("base64url")}`;
}
```

---

## 3 — Repository (`apps/backend/src/db/repositories/oauth.repo.ts`)

Add to the existing `oauthRepository` object:

```ts
async getRefreshToken(token: string): Promise<OAuthRefreshTokenRow | null>
async setRefreshToken(token: string, data: {
  client_id: string;
  user_id: string;
  scope: string;
  access_token: string | null;
  expires_at: number; // ms epoch
}): Promise<void>
async deleteRefreshToken(token: string): Promise<void>
// Atomic rotate: insert new + delete old in one transaction.
async rotateRefreshToken(oldToken: string, newToken: string, data: {
  client_id: string;
  user_id: string;
  scope: string;
  access_token: string | null;
  expires_at: number;
}): Promise<void>
```

Use `db.transaction(async (tx) => { ... })` for `rotateRefreshToken`. Extend the existing `cleanupExpired()` to also `DELETE FROM oauth_refresh_tokens WHERE expires_at < NOW()`.

---

## 4 — Token endpoint (`apps/backend/src/routers/oauth/token.ts`)

### 4a. Env-driven TTLs (top of handler)

```ts
const ACCESS_TTL = parseInt(process.env.OAUTH_ACCESS_TOKEN_TTL_SECONDS ?? "3600", 10);
const REFRESH_TTL = parseInt(process.env.OAUTH_REFRESH_TOKEN_TTL_SECONDS ?? "2592000", 10);
```

### 4b. Replace lines 36-41 with a grant_type switch

```ts
if (grant_type === "authorization_code") {
  // existing flow (lines 43-190) — see 4c
} else if (grant_type === "refresh_token") {
  // new flow — see 4d
} else {
  return res.status(400).json({
    error: "unsupported_grant_type",
    error_description: "Supported grant types: authorization_code, refresh_token",
  });
}
```

### 4c. `authorization_code` branch — extend the response

After the access token is generated and stored (around line 183), also:

```ts
const refreshToken = generateSecureRefreshToken();
await oauthRepository.setRefreshToken(refreshToken, {
  client_id: codeData.client_id,
  user_id: codeData.user_id,
  scope: codeData.scope,
  access_token: accessToken,
  expires_at: Date.now() + REFRESH_TTL * 1000,
});

res.json({
  access_token: accessToken,
  refresh_token: refreshToken,
  token_type: "Bearer",
  expires_in: ACCESS_TTL,
  scope: codeData.scope,
});
```

Replace `expiresIn = 3600` with `ACCESS_TTL` throughout.

### 4d. `refresh_token` branch — new

```ts
const { refresh_token, client_id } = req.body;
if (!refresh_token) return 400 invalid_request;

const rtData = await oauthRepository.getRefreshToken(refresh_token);
if (!rtData) return 400 invalid_grant;
if (Date.now() > rtData.expires_at.getTime()) {
  await oauthRepository.deleteRefreshToken(refresh_token);
  return 400 invalid_grant ("refresh token expired");
}
if (rtData.revoked_at) return 400 invalid_grant ("refresh token revoked");
if (rtData.client_id !== client_id) return 400 invalid_grant;

// Authenticate the client (basic / post / none) — extract existing logic to helper
const clientData = await oauthRepository.getClient(client_id);
if (!clientData) return 400 invalid_client;
authenticateClient(req, clientData);  // throws → respond 401

// Mint new access token + new refresh token, rotate atomically
const newAccess = generateSecureAccessToken();
const newRefresh = generateSecureRefreshToken();
await oauthRepository.setAccessToken(newAccess, {
  client_id: rtData.client_id,
  user_id: rtData.user_id,
  scope: rtData.scope,
  expires_at: Date.now() + ACCESS_TTL * 1000,
});
await oauthRepository.rotateRefreshToken(refresh_token, newRefresh, {
  client_id: rtData.client_id,
  user_id: rtData.user_id,
  scope: rtData.scope,
  access_token: newAccess,
  expires_at: Date.now() + REFRESH_TTL * 1000,
});

res.json({
  access_token: newAccess,
  refresh_token: newRefresh,
  token_type: "Bearer",
  expires_in: ACCESS_TTL,
  scope: rtData.scope,
});
```

### 4e. Extract `authenticateClient(req, clientData)` helper

Lines 96-129 today live inside the `authorization_code` branch and run client-secret validation. Move that block to a helper (file-local function or to `utils.ts`) so both grant branches can call it without duplication. It should `return { ok: true }` or `return { ok: false, status, body }` and the handler responds accordingly — keep the response shape identical to today to avoid behavior drift.

---

## 5 — Zod types (`packages/zod-types/src/oauth.zod.ts`)

Add a schema mirroring `OAuthAccessTokenSchema` for the refresh-token row, plus update any token-response schema that currently lacks `refresh_token` if applicable. (`OAuthTokensSchema.refresh_token` is already optional at line 17 — frontend will pick it up automatically.)

---

## 6 — Env documentation (`example.env`)

Append:

```
# OAuth token lifetimes (seconds). Defaults: 1h access, 30d refresh.
# OAUTH_ACCESS_TOKEN_TTL_SECONDS=3600
# OAUTH_REFRESH_TOKEN_TTL_SECONDS=2592000
```

---

## Verification

Run the backend with a short access TTL to make the full cycle observable in <2 min:

```bash
OAUTH_ACCESS_TOKEN_TTL_SECONDS=60 pnpm dev
```

Then end-to-end via curl (replace `$BASE`, capture each token from the previous response):

```bash
# 1. Auth-code flow → expect access_token AND refresh_token in response
curl -sX POST $BASE/oauth/token \
  -H 'Content-Type: application/json' \
  -d "{\"grant_type\":\"authorization_code\",\"code\":\"$CODE\",\"client_id\":\"$CID\",\"redirect_uri\":\"$RU\",\"code_verifier\":\"$CV\"}"
# Expect: { access_token, refresh_token, token_type: "Bearer", expires_in: 60, scope }

# 2. Use access token against any protected MCP endpoint (introspect is convenient)
curl -sX POST $BASE/oauth/introspect -H 'Content-Type: application/json' \
  -d "{\"token\":\"$ACCESS\"}"
# Expect: { active: true, ... }

# 3. After 60s, access token expires
sleep 65
curl -sX POST $BASE/oauth/introspect -H 'Content-Type: application/json' \
  -d "{\"token\":\"$ACCESS\"}"
# Expect: { active: false }

# 4. Refresh
curl -sX POST $BASE/oauth/token -H 'Content-Type: application/json' \
  -d "{\"grant_type\":\"refresh_token\",\"refresh_token\":\"$REFRESH\",\"client_id\":\"$CID\"}"
# Expect: NEW access_token, NEW refresh_token

# 5. Confirm rotation: old refresh token must fail
curl -sX POST $BASE/oauth/token -H 'Content-Type: application/json' \
  -d "{\"grant_type\":\"refresh_token\",\"refresh_token\":\"$REFRESH_OLD\",\"client_id\":\"$CID\"}"
# Expect: 400 invalid_grant

# 6. Regression — unsupported grant
curl -sX POST $BASE/oauth/token -H 'Content-Type: application/json' \
  -d '{"grant_type":"password","username":"x","password":"y"}'
# Expect: 400 unsupported_grant_type
```

Then real-world: connect claude.ai's connector, leave it idle ≥1 hour, return and confirm it still works without a re-auth prompt.

---

## Risks & rollback

- **Existing access tokens have no associated refresh token.** Acceptable — they expire within the access TTL and clients re-auth once. No backfill needed.
- **Concurrent refresh from same client (race).** `rotateRefreshToken` uses a DB transaction; the second concurrent request will hit `getRefreshToken` after the row is gone and return `invalid_grant`. v1.1 reuse-detection turns that into chain-invalidation per OAuth 2.1 BCP §4.13.
- **FK cascade.** `oauth_refresh_tokens.access_token` uses `ON DELETE SET NULL` so deleting an access token does not delete its refresh token (a refresh token's purpose is to outlive access tokens).
- **Rollback.** Revert the changes to schema.ts, utils.ts, oauth.repo.ts, token.ts, oauth.zod.ts. Drop the table: `DROP TABLE oauth_refresh_tokens;`. The migration is purely additive — no other table is touched.

---

## Reused existing code

- `apps/backend/src/routers/oauth/utils.ts` — `randomBytes`-based generator pattern (mirror it for refresh)
- `apps/backend/src/db/repositories/oauth.repo.ts` — existing CRUD shape and `cleanupExpired` (extend it)
- `apps/backend/src/routers/oauth/token.ts` — existing PKCE / client-auth blocks (extract auth block to helper, reuse in refresh branch)
- `packages/zod-types/src/oauth.zod.ts` — `OAuthTokensSchema.refresh_token` already optional (no change needed there)
- `apps/backend/src/routers/oauth/metadata.ts` — already advertises `refresh_token` grant; this fix makes that truthful
