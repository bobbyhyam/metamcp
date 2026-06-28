# OAuth refresh-token improvements over upstream #276

Context: our fork's PR [#313](https://github.com/metatool-ai/metamcp/pull/313)
(`oauth: implement refresh token issuance and rotation`) was closed as superseded
by upstream [#276](https://github.com/metatool-ai/metamcp/pull/276), which was
merged into `ai-dev`. #276 fully covers the core feature (Claude.ai connectors can
silently refresh the 1h access token), so the feature itself is not missing
upstream.

However, two improvements from our #313 did **not** land in #276. This document
records them so they can be re-applied on top of the merged upstream code. Each is
tracked by a fork issue:

- Section 1 → [bobbyhyam/metamcp#3](https://github.com/bobbyhyam/metamcp/issues/3)
- Section 2 → [bobbyhyam/metamcp#4](https://github.com/bobbyhyam/metamcp/issues/4)

---

## 1. Atomic refresh-token rotation (transaction-wrapped)

**What upstream #276 does:** rotation is two separate awaits — delete the old token
row, then insert the new pair:

```ts
// #276 handleRefreshTokenGrant
await oauthRepository.deleteAccessToken(tokenData.access_token);
const { accessToken, refreshToken } = await issueTokenPair(/* ... */);
```

There is a window between the delete and the insert. If the process dies or the
insert fails after the delete commits, the client is left holding a refresh token
that no longer exists server-side and must perform a full re-authentication.

**What our #313 does:** rotation is a single DB transaction — insert the new token
then delete the old one atomically, so it either fully succeeds or fully rolls
back. From `apps/backend/src/db/repositories/oauth.repo.ts`:

```ts
async rotateRefreshToken(oldToken, newToken, data) {
  await db.transaction(async (tx) => {
    await tx.insert(oauthRefreshTokensTable).values({
      refresh_token: newToken,
      client_id: data.client_id,
      user_id: data.user_id,
      scope: data.scope,
      access_token: data.access_token,
      expires_at: new Date(data.expires_at),
    });
    await tx
      .delete(oauthRefreshTokensTable)
      .where(eq(oauthRefreshTokensTable.refresh_token, oldToken));
  });
}
```

**Port plan onto #276:** wrap #276's delete-old + insert-new sequence in
`db.transaction(...)`. Because #276 stores refresh tokens as nullable columns on
`oauth_access_tokens` (rather than a dedicated table), the transaction body becomes
delete-old-row + insert-new-row on `oauthAccessTokensTable`, ordered insert-first
then delete to match the rollback guarantee.

---

## 2. Env-tunable token TTLs (with a longer refresh default)

**What upstream #276 does:** TTLs are hardcoded constants:

```ts
// #276 token.ts
const ACCESS_TOKEN_EXPIRY = 3600;          // 1 hour
const REFRESH_TOKEN_EXPIRY = 7 * 24 * 3600; // 7 days
```

A deployment cannot change these without editing source. A connector left idle
longer than 7 days forces a full re-auth.

**What our #313 does:** both TTLs are read from the environment, with the refresh
default raised to 30 days. From `apps/backend/src/routers/oauth/token.ts`:

```ts
const ACCESS_TTL  = parseInt(process.env.OAUTH_ACCESS_TOKEN_TTL_SECONDS  ?? "3600", 10);
const REFRESH_TTL = parseInt(process.env.OAUTH_REFRESH_TOKEN_TTL_SECONDS ?? "2592000", 10); // 30d
```

Documented in `example.env`:

```sh
# OAUTH_ACCESS_TOKEN_TTL_SECONDS=3600
# OAUTH_REFRESH_TOKEN_TTL_SECONDS=2592000
```

**Port plan onto #276:** replace #276's two `const ..._EXPIRY` constants with the
`parseInt(process.env... ?? default)` reads, thread them through `issueTokenPair`
and the `expires_in` response field, and add the two commented vars to
`example.env`.

---

## Notes / not porting

- A third difference — dedicated `oauth_refresh_tokens` table with `replaced_by` /
  `revoked_at` columns for OAuth 2.1 BCP §4.13 reuse detection — is **not** being
  ported. It was forward-compat scaffolding only (never wired up), and #276's
  nullable-column approach is the one now merged upstream. Re-introducing a
  separate table would mean fighting the upstream schema for no current benefit.
