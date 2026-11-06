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
  entry at this stage is normal. `[verified 2026-11-06 — bare probe 429; error → installed after OAuth]`
- **The repair verb is `capabilities_refresh dropbox`.** Never re-run
  `capabilities_add` to fix a broken connection — it rewrites the registry
  entry with `auth.type: none` and can destroy the auth config
  (`openpave-wrike-mcp` doctrine, same bridge). `[PAVE source + wrike precedent]`
- After connect, set `auth.type: "oauth2"` on the `servers.json` entry
  (read-modify-write; `capabilities_add` leaves it `none`), then run the
  OAuth start below. `[verified 2026-11-06]`

### R10 first-use grant — the enum trap

Bridged calls are refused with `permissionAsk: true` until
`mcp:dropbox` is allowed. The server-side `permission` tool supports
`type: "mcp"` (`tools/permission.js:1270`), but **the agent-surface tool
schema enum omits `mcp` and the client coerces the value to `module`**,
which writes a useless `modules` grant. `[verified 2026-11-06 — three coerced
calls; modules:* already present made them all no-ops]`

Working paths, in order: `[verified 2026-11-06]`

1. `permission` tool `action: "grant"`, `type: "mcp"`, `target: "mcp:dropbox"`
   — works from surfaces whose schema passes `mcp` through; shows the user
   a confirmation card.
2. Direct edit of `~/.pave/permissions.yaml` (read-modify-write):

   ```yaml
   permissions:
     mcp:
       allowed:
         - mcp:dropbox
   ```

   `mcp-config-store.js` re-reads on mtime change — no restart needed.
   `[PAVE source — mcp-config-store.js:260 mcpPermissionDecision, mtime cache]`

## 2. Auth

Dropbox's MCP surface is **OAuth-only** — no API key, no M2M/client-credentials
flow; every token traces to a user's browser consent. `[web]`

**Outcome 1 is live: DCR accepted, full OAuth lane works.**
`[verified 2026-11-06]` Discovery chain `www-authenticate+prm+as`
(RFC 9728 PRM via the `WWW-Authenticate` header), RFC 7591 DCR issued PAVE
its own client id (`ydww2fwnzkxganl`), PKCE S256, RFC 8707 resource parameter,
scopes `account_info.read files.* file_requests.* sharing.*`. Tokens land as
`MCP__dropbox__ACCESS/REFRESH` in `~/.pave/tokens.yaml`; the entry ends
`status: installed` with `audienceOrigin: https://mcp.dropbox.com` pinned.
The "trusted DCR client list" restriction from third-party writeups did not
apply to the live server. `[web claim contradicted by direct test]`

The other two outcomes, kept for fallback if Dropbox changes behavior:

2. **DCR refused** → pin your own app: register in the Dropbox App Console,
   set `auth.oauth.clientId` (+ `authorizeUrl`/`tokenUrl` pins) on the
   `servers.json` entry. Resolution order: persisted DCR id > registry pin >
   fresh DCR. `[PAVE source — mcp-oauth-flow.js:402-403]`
3. **Confidential client demanded** → PAVE refuses the flow before minting
   state (#3110 guard) and names the bearer path. Bearer is a poor fit:
   Dropbox access tokens are short-lived (~4 h). `[PAVE source — mcp-oauth-flow.js:397-400]` `[web — token lifetime]`

Agent-driven OAuth start (no browser session on the agent side needed — the
user opens the `authorizeUrl` anywhere):

```bash
curl -s -X POST http://127.0.0.1:4096/capabilities/mcp/dropbox/oauth/start \
  -H "x-pave-internal-token: $(cat ~/.pave/.internal-token)" \
  -H "x-pave-internal-user: <owner email>" \
  -H 'Content-Type: application/json' -d '{}'
```

Sbx note: unlike the CLI skill's brokered OAuth (local-only, 403 behind the
sbx gateway), **MCP OAuth start and bridged execution both pass the pinned
sbx member**. `[PAVE source — #3106 arms: mcp-oauth-flow.js:338, mcp-manager.js:784]`

## 3. Tools — 45, snapshot-verified

`[verified 2026-11-06 — live tools/list snapshot in servers.json, all 45 rows
carry inputSchema; e2e calls who_am_i / get_usage_and_quota / list_folder
succeeded through the bridge]` Third-party writeups said ~23 file tools;
the live surface is larger — Paper and Sign are included.

| Group | Tools |
|---|---|
| Core files (15) | `who_am_i`, `get_usage_and_quota`, `list_folder`, `search`, `get_file_metadata`, `get_file_content`, `create_file`, `create_folder`, `copy`, `move`, `delete`, `download_link`, `list_file_revisions`, `restore_file_revision`, `get_file_content_metadata` |
| Sharing (3) | `create_shared_link` (private view-only; **cannot** make public/anyone-links — `INVALID_AUDIENCE`), `get_shared_link_metadata`, `list_shared_links` |
| File requests (3) | `create_file_request`, `get_file_request`, `list_file_requests` |
| History/rewind (2) | `list_restore_events`, `restore_folder` |
| Extraction/transform (2) | `run_file_content_extraction` (markdown/text/OCR/transcript/keyframes), `transform_file` (PDF/HTML convert, thumbnails, video frames — nothing written to Dropbox) |
| Catalog bridge (4) | `list_dropbox_operations`, `get_dropbox_operation`, `run_read_dropbox_operation`, `run_write_dropbox_operation` — full Dropbox API surface via operation ids; MUST `get_dropbox_operation` before `run_*` |
| Async (1) | `check_job_status` — polls move/copy/delete/restore_folder jobs |
| Paper (10) | `paper_list_recent_docs`, `paper_read_document`, `paper_create_document`, `paper_insert`, `paper_insert_image`, `paper_edit_content`, `paper_edit_table`, `paper_apply_markdown`, `paper_comment`, `paper_get_doc_history` |
| Sign (5) | `sign_list_signature_requests`, `sign_create_signature_request_draft`, `sign_download_signature_request`, `sign_send_signature_request_reminders`, `sign_cancel_signature_request` |

Dump the live surface yourself:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema <tool>
```

## 4. Path & ID doctrine

`[verified 2026-11-06 — live schemas + live list_folder/who_am_i responses]`

- **Three identifier forms** accepted by path params: fq_path (`/Reports/q3.pdf`),
  ns_path (`ns:14397394131//Reports`), file_id (`id:yFBSsv4HM_...`), plus
  `id:<id>/child/path` composites. Dropbox shared/web URLs are NOT accepted.
- **Responses return ns_path form** (`ns:N//name`), not fq_path — keep what a
  call returns and feed it back verbatim; don't hand-convert between forms.
- `who_am_i` gives the namespace map: `root_namespace_id` ≠
  `home_namespace_id` on team accounts (root = team space, home = member
  folder). Mounted team folders appear in listings as `object_type: "mount"`.
- `object_type` values: `mount` | `folder` | `file`. Paper docs list with a
  `.paper` extension and a placeholder `size: 200` — that is not real bytes.
- **`list_folder` is RECURSIVE BY DEFAULT** (`recursive: true or omitted
  lists descendants recursively`). An unbounded `{"path": ""}` on a real
  account returns the whole tree — pass `recursive: false` and/or
  `max_results` (1–600) unless you mean it. `[verified 2026-11-06 — root call
  returned nested tree across 4 namespaces]`
- Pagination is cursor-based (`cursor` in, `cursor` out) on `list_folder`,
  `search`, `list_shared_links`, `list_file_requests`, `list_restore_events`.

## 5. Diagnostic fallback: drive the server directly

The bridge can swallow real error text (wrike bridge defect §0A; re-check
here). When a bridged call fails with a bare `isError`, use the committed
probe — it is in THIS repo, so unlike the wrike probe it cannot be wiped with
`~/.pave/cache`:

```bash
node probe/dropbox-probe.mjs tools/list
node probe/dropbox-probe.mjs schema get_file_metadata
node probe/dropbox-probe.mjs call search '{"query":"invoice","max_results":10}'
```

The probe reads the token from `~/.pave/tokens.yaml` — never from argv, never
echoed. Behind a proxy, prefix with `NODE_USE_ENV_PROXY=1`.

Bridge behavior observed so far: results arrive as a raw JSON string payload
(the MCP layer stringifies structured content). Parse it; don't expect pretty
objects. `[verified 2026-11-06 — who_am_i/quota/list_folder all stringified]`

## 6. Config reference

| Key | Where | Purpose | Status |
|---|---|---|---|
| `MCP__dropbox__ACCESS` | `~/.pave/tokens.yaml` | MCP access token (0600) | `[verified 2026-11-06]` |
| `MCP__dropbox__REFRESH` | `~/.pave/tokens.yaml` | MCP refresh token | `[verified 2026-11-06]` |
| `auth.type` | `~/.pave/mcp/servers.json` | `oauth2` (OAuth path) or `bearer` (fallback) | `[verified 2026-11-06 — oauth2 live]` |
| `oauth.registeredClientId` | `~/.pave/mcp/servers.json` | DCR-issued client id (`ydww2fwnzkxganl`) | `[verified 2026-11-06]` |
| `audienceOrigin` | `~/.pave/mcp/servers.json` | R6 pin — token never replays to another origin | `[verified 2026-11-06 — set after callback]` |
| `mcp:dropbox` | `~/.pave/permissions.yaml` → `permissions.mcp.allowed` | R10 first-use approval for the bridge | `[verified 2026-11-06 — grant unblocked who_am_i]` |

## 7. Open questions

1. Does the doubled-result bridge defect (wrike §0B) reproduce here?
   `[untested]`
2. Does the swallowed-error-text bridge defect (wrike §0A) reproduce here?
   `[untested]`
3. Does the token refresh lane work unattended past the first ~4 h access
   token expiry (no forced re-consent)? `[untested — first e2e ran inside
   one token lifetime]`

Closed 2026-11-06: PRM discovery + public-client AS admission (was Q1) —
YES, via `www-authenticate`; DCR acceptance (was Q2) — YES, own client id
issued; live tool surface (was Q3) — 45 tools, table in §3.
