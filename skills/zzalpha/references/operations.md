# AlphaDB Operations — Auth, As-Of, Deployment Verification

## Authentication — three tiers

All passed via `Authorization: Bearer <key>` on the HTTP API; MCP clients send them via `X-API-Key` env-forwarded header.

| Key | Env Var | Access |
|-----|---------|--------|
| **API Key** | `ALPHA_API_KEY` | All read-only `/v1/*` routes + as-of-date GET |
| **Admin Key** | `ALPHA_ADMIN_KEY` | All mutations: symbol tracking, batch track/untrack, job cancel/retry, rates fetch/backfill, analytics enable/disable, sync trigger, provider switch, as-of-date set/clear |
| **As-Of Key** | `ALPHA_AS_OF_KEY` | As-of-date GET/set/clear only |

**All mutation endpoints require the Admin Key.** Read-only GETs accept the API Key or no auth depending on `REQUIRE_AUTH` config.

## MCP client key forwarding

The MCP server sends `X-API-Key` on every HTTP request to the gateway. Trading tools use `ALPHA_OMS_KEY` if set, falling back to `ALPHA_API_KEY`.

- Paper and sim trading (paper, schwab-sim, alpaca accounts) require only `ALPHA_API_KEY`.
- The OMS key check is **per-account**: only `provider=schwab` live accounts require `ALPHA_OMS_KEY`.

If no keys are configured on the gateway, the env vars can be omitted.

## Verifying a deployed binary

When a client (AlphaSeeker) reports that a feature "doesn't work" against the deployed server, **always disambiguate code bug vs stale deployment before reading the evaluator**.

The alphaserver binary on the LXC is not stripped, so `grep -a` on the ELF finds code markers directly — no `go version -m` needed:

```bash
ssh alpha-lxc 'grep -aoE "DELTA_BREACH|GAMMA_RISK|NeedsGreeks|QuoteAtUnderlying|<your-symbol>" /opt/alphadb/alphaserver | sort -u'
```

## Pid vs disk binary

Also check the running pid and its mapped binary:

```bash
ssh alpha-lxc 'ls -la /proc/$(pgrep alphaserver)/exe'
```

This shows whether the process is still running an older in-memory binary that was since replaced on disk. **Clients see the mapped-in-memory version, not the file on disk.**

## Version tag freshness

The version tag reported in client handshake logs — `AlphaDB Server connected: ... (vDEADBEEF, ...)` — is load-bearing for replay test validity. If the tag is stale or frozen across restarts, clients will tune strategies against a binary whose version number is lying. **Verify tag freshness before trusting any feature-flag test result.**

## Database connection

- Prod LXC at `10.1.5.30`, prod DB on port `5432`, user `alphadb`.
- SSH via `alpha-lxc` (config shortcut).
- OMS data is in a **separate database** `alphadb_oms`, not `alphadb`. Always use `OMS_DB_URL`.

## Local test environment

```bash
make infra-up
export TEST_DB_URL=postgres://medoya:medoya@localhost:5434/medoya
export TEST_REDIS_URL=localhost:6379
make test-unit
```

- TimescaleDB on port 5434
- Redis on port 6379
- `make test-unit` uses `-short` and skips testcontainers
- `make test-integration` runs integration tests (real DB via testcontainers or `TEST_DB_URL`)

## As-of-date administration

```
POST   /v1/admin/as-of-date   { "date": "2026-02-14" }   ← set (Admin or As-Of key)
DELETE /v1/admin/as-of-date                              ← clear (Admin or As-Of key)
GET    /v1/admin/as-of-date                              ← read (any key)
```

See `oms_replay.md` for the replay determinism invariants that `asOfBoundary` enforces.

## Never run destructive prod ops without approval

Never `DELETE`/`UPDATE` production data without explicit user approval. Ask first, explain consequences. This applies to both the main `alphadb` and OMS `alphadb_oms` databases.

## Verify claims against the live database

Don't trust second-hand explanations about what's in the database. Query the actual data before confirming behavior. Memory or docs may be stale; the DB is the truth.
