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
pave install cnrai/openpave-dropbox-mcp
```

Then read the doctrine from any PAVE session:

```
skill_docs(skill="dropbox-mcp")
```

### Orchestrated installs

Users can also say `install https://github.com/cnrai/openpave-dropbox-mcp`
to any live PAVE session (A). A does not run the setup runbook inline: it
pre-registers the MCP server (the one carded step), spawns a dedicated
installer session (B) from this repo's committed brief
([orchestration/brief.template.md](orchestration/brief.template.md)), and
relays every user question between B and the user — B has no user surface
of its own. The full doctrine, wire protocol, and failure paths are
SKILL.md §10.

## No formatting skill

**This skill has no formatting skill.** It ships no converter, renderer,
template, or styling layer, and it does not lay out, brand, or restyle a
document for you.

- Paper edits are markdown applied to a document that already exists —
  `paper_edit_content`, `paper_edit_table`, `paper_apply_markdown`,
  `paper_insert`. The output looks like whatever markdown the caller wrote.
- `run_file_content_extraction` (markdown/text/OCR/transcript/keyframes) and
  `transform_file` (PDF/HTML convert, thumbnails, video frames) are Dropbox's
  own server-side extraction and conversion tools: they return text or emit a
  new file. They do not style or lay out one.
- `create_file` writes UTF-8 text only — it cannot build a docx, pdf, or xlsx.

Produce formatted output with a dedicated formatting skill (e.g.
`openpave-pdf` for branded PDFs, `openpave-deck` for slide decks), then store
or move the finished artifact with this skill.

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
