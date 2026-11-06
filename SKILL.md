# Dropbox MCP Skill Doctrines (SKILL.md)

Doctrines for working with Dropbox from PAVE through the **remote Dropbox MCP
server** (`https://mcp.dropbox.com/mcp`, bridged as `mcp_dropbox_*` tools).

Every claim carries a tag. Trust them differently:

| Tag | Meaning |
|---|---|
| `[verified <date>]` | Tested by PAVE against the live server/account. |
| `[PAVE source]` | Read directly from openpave server source (line-cited). Behavior of PAVE, not of Dropbox. |
| `[web]` | From Dropbox help pages or third-party writeups (usecarly.com, scalekit.com, 2026-07). Not independently reproduced. |
| `[untested]` | No evidence either way. Do not rely on it. |

If you change something in this file, replace blanket dates with per-claim
tags. A single "verified" stamp over unverified material launders guesswork
into fact.

---

## 0. What this skill is — and is not

- Doctrine-only skill: entrypoint is this file, no local code, **no sandbox
  applies**. The Dropbox MCP is a remote HTTPS service; no Dropbox code runs
  on your machine. `[verified 2026-11-06 — package layout + PAVE MCP architecture]`
- The MCP token never enters a skill sandbox or process env: openpave R6
  attaches it only inside the PAVE server's `connectServer()`.
  `[PAVE source — server/mcp-manager.js R6, lines 10-13]`
- **The CLI skill's credential does not transfer.** The sandboxed `dropbox`
  skill's `DROPBOX_REFRESH_TOKEN` belongs to the CNR broker's OAuth app; the
  MCP server grants only its own app. Never paste one into the other's slot.
  `[verified by construction — two different OAuth client registrations]`

## 1. Installing the Dropbox MCP in PAVE

The remote server is Streamable-HTTP at `https://mcp.dropbox.com/mcp`.
`[web]` Open beta since March 2026. `[web]`

```
capabilities_add https://mcp.dropbox.com/mcp
```

- PAVE derives the entry id from the host: `mcp.dropbox.com` → `dropbox`,
  so bridged tools are named `mcp_dropbox_*` and the token key is
  `MCP__dropbox__ACCESS`. `[PAVE source — capabilities-custom-routes.js host→id rule]`
- Expect first connect to end `status: error` until auth completes
  (`401` unauthorized, or `429` — the edge rate-limits bare probes;
  observed on `GET /` and `/.well-known/...` 2026-11-06). A dead-looking
  entry at this stage is normal. `[verified 2026-11-06 — bare probe 429]`
- **The repair verb is `capabilities_refresh dropbox`.** Never re-run
  `capabilities_add` to fix a broken connection — it rewrites the registry
  entry with `auth.type: none` and can destroy the auth config
  (`openpave-wrike-mcp` doctrine, same bridge; re-verify here after any
  re-add). `[PAVE source + wrike precedent]`

## 2. Auth

Dropbox's MCP surface is **OAuth-only** — no API key, no M2M/client-credentials
flow; every token traces to a user's browser consent. `[web]`

Three possible outcomes for PAVE (a public PKCE client with no secret,
`token_endpoint_auth_method: none` `[PAVE source — mcp-oauth-flow.js:301-311]`):

1. **RFC 7591 DCR accepted** → full OAuth works through PAVE's MCP OAuth lane
   (PRM discovery → DCR → PKCE → `MCP__dropbox__ACCESS/REFRESH` in
   `~/.pave/tokens.yaml`, audience-bound to `https://mcp.dropbox.com`).
   `[PAVE source — mcp-oauth-flow.js:357-423]`
2. **DCR refused** (Dropbox restricts DCR to a trusted client list — Claude
   Code/Web, ChatGPT Web/Codex, Cursor; PAVE is not on it `[web]`) →
   fall back to a pinned client id: register your own app in the Dropbox App
   Console and set `auth.oauth.clientId` (+ `authorizeUrl`/`tokenUrl` pins) on
   the `servers.json` entry. Resolution order: persisted DCR id > registry
   pin > fresh DCR. `[PAVE source — mcp-oauth-flow.js:402-403]`
3. **Authorization server demands a confidential client** → PAVE refuses the
   OAuth flow before minting state (#3110 guard) and names the bearer path.
   Bearer is a poor fit here: Dropbox access tokens are short-lived (~4 h),
   so a bearer entry needs manual re-tokening — prefer the pinned-client OAuth
   path. `[PAVE source — mcp-oauth-flow.js:397-400]` `[web — token lifetime]`

Sbx note: unlike the CLI skill's brokered OAuth (local-only, 403 behind the
sbx gateway), **MCP OAuth start and bridged execution both pass the pinned
sbx member**. `[PAVE source — #3106 arms: mcp-oauth-flow.js:338, mcp-manager.js:784]`

## 3. Tools

`[untested — table to be filled from live tools/list during e2e install; ~20-23
file-centric tools expected: list/search/read content/create (≤5 MB inline)/
copy/move/delete, shared links, file requests, revisions. No team admin, no
webhooks, no large-upload sessions. [web]]`

Dump the live surface yourself:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema <tool>
```

## 4. Path & ID doctrine

`[untested — Dropbox MCP is expected to take display paths (e.g. /Reports/q3.pdf)
rather than canonical ids; confirm from live schemas before writing routing logic.]`

## 5. Diagnostic fallback: drive the server directly

The bridge can swallow real error text (wrike bridge defect §0A; re-check
here). When a bridged call fails with a bare `isError`, use the committed
probe — it is in THIS repo, so unlike the wrike probe it cannot be wiped with
`~/.pave/cache`:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema get_file_metadata
node probe/dropbox-probe.mjs call search_files '{"query":"invoice","max_results":10}'
```

The probe reads the token from `~/.pave/tokens.yaml` — never from argv, never
echoed. Behind a proxy, prefix with `NODE_USE_ENV_PROXY=1`.

## 6. Config reference

| Key | Where | Purpose | Status |
|---|---|---|---|
| `MCP__dropbox__ACCESS` | `~/.pave/tokens.yaml` | MCP access token (0600) | `[untested — key shape expected from R6 naming]` |
| `MCP__dropbox__REFRESH` | `~/.pave/tokens.yaml` | MCP refresh token (OAuth path only) | `[untested]` |
| `auth.type` | `~/.pave/mcp/servers.json` | `oauth2` (OAuth path) or `bearer` (fallback) | `[untested]` |
| `audienceOrigin` | `~/.pave/mcp/servers.json` | R6 pin — token never replays to another origin | `[PAVE source]` |
| `mcp:dropbox` | `~/.pave/permissions.yaml` | R10 first-use approval for the bridge | `[PAVE source]` |

## 7. Open questions

1. Does `mcp.dropbox.com` publish RFC 9728 protected-resource metadata, and
   does its AS admit public clients (`token_endpoint_auth_methods_supported`
   including `none`)? Decides path 1 vs 2 in §2. `[untested — 429 hid the answer 2026-11-06]`
2. Does DCR accept PAVE despite the trusted list? `[untested]`
3. Exact tool names/schemas — §3 table is a placeholder until a live
   `tools/list` is captured. `[untested]`
4. Does the doubled-result bridge defect (wrike §0B) reproduce here?
   `[untested]`
5. Does the swallowed-error-text bridge defect (wrike §0A) reproduce here?
   `[untested]`
