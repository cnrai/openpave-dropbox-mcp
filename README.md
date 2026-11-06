# dropbox-mcp

Doctrine skill for working with Dropbox from PAVE via the remote **Dropbox
MCP server** (`https://mcp.dropbox.com/mcp`, bridged as `mcp_dropbox_*`
tools).

This is a doctrine-only skill: it ships no local CLI. Its payload is
[`SKILL.md`](SKILL.md) — verified operational knowledge for the Dropbox MCP
bridge, plus a committed diagnostic probe ([`probe/dropbox-probe.mjs`](probe/dropbox-probe.mjs))
for reading real server error messages the bridge swallows and for dumping
live tool schemas.

## Install

```bash
pave install johnmak101-ops/openpave-dropbox-mcp
```

Then read the doctrine from any PAVE session:

```
skill_docs(skill="dropbox-mcp")
```

## Relationship to the `dropbox` CLI skill

The sandboxed `dropbox` skill runs local JS against the Dropbox REST API
with its own OAuth app and a sandboxed auth proxy. This skill is the
MCP-bridged counterpart: no local code, no sandbox — everything goes through
Dropbox's hosted MCP server. The two credentials are **not interchangeable**
(different OAuth apps; see SKILL.md).

## Why MCP instead of the CLI skill

- No sandbox, no token plumbing into skill processes (R6: the token is
  attached only inside the PAVE server's `connectServer()`).
- Works from pinned sandbox (sbx) member sessions: both the MCP OAuth start
  and bridged tool execution pass the #3106 member arm — unlike the CLI
  skill's brokered OAuth, which is local-only and 403s behind the sbx
  gateway.

## License

MIT
