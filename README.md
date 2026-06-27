# Sharedrop Agent Skill

The official agent skill for [**Sharedrop**](https://sharedrop.cloud) — the sharing
layer for the agent economy.

```
npx skills add ShareDropCloud/skills --skill sharedrop -g
```

## What is Sharedrop?

Sharedrop is the sharing layer for the agent economy: the easy way to get what an AI
generates in front of the people (and other agents) who need to see it, safely, with a
stable link you control. One upload, one durable URL, the right people get in.

- **Agents upload** via the **`sharedrop` CLI** (the primary surface), a hosted MCP
  server, or the REST API — streamed, any file type, no base64. A `SHAREDROP_TOKEN`
  env var is all an agent runtime needs.
- **Humans read** at a stable URL in any browser. Re-upload with the same page id to
  revise it — the URL never changes and a version is recorded, so "update that report"
  never breaks anyone's link.
- **Agents read back, too.** `fetch` pulls a page's raw content into an agent's context
  on every tier; `download` zips the whole artefact for a human.
- **The right people get access.** Pages default to private. Invite people by email —
  they sign in with that email to view (account-free recipient access is *coming soon*).
  Publish publicly on explicit request, or (Pro) mint disappearing links that expire by
  time or view count.
- **Untrusted content stays contained.** Uploaded HTML is sanitised and rendered in a
  locked-down sandbox on a separate origin with strict CSP — interactive pages run
  JavaScript only when fully self-contained.

More: [docs](https://sharedrop.cloud/docs) · [llms.txt](https://sharedrop.cloud/llms.txt)
· [pricing](https://sharedrop.cloud/pricing)

## What is an agent skill?

A skill is a markdown instruction file (`SKILL.md`) that teaches an AI agent *when* and
*how* to use a capability — judgment, not plumbing. Here the plumbing is the **`sharedrop`
CLI** (with the hosted MCP server and REST API as fallbacks); the skill makes the agent
reach for the right surface and use it well.

## The skill in this repo

### `sharedrop` — [skills/sharedrop/SKILL.md](skills/sharedrop/SKILL.md)

Teaches an agent to share generated documents through Sharedrop instead of pasting walls
of HTML into chat or attaching files. It **leads with the CLI** as the best agent surface
and falls back to MCP/REST only when there's no shell. Specifically:

- **When to upload** — any time it produces a report, dashboard, summary, or other
  rendered artifact the user should see, archive, link to, or hand to someone else; and
  when *not* to (secrets, anything the user didn't ask to publish).
- **The CLI** — `sharedrop upload` (a file or piped stdin), `update <id>` to revise a
  page in place, `fetch`/`download` to read a page back, `share --email`, `list`/`get`.
  Authenticate once with `sharedrop login` or `SHAREDROP_TOKEN`.
- **Stable URLs** — re-upload with the same page id so iterative work keeps one link and
  accumulates version history, instead of minting duplicate pages.
- **Sensible visibility** — default `private`; `public` only on explicit request; share
  with specific people by email; disappearing links on Pro.
- **Self-contained interactive pages** — interactive HTML runs scripts only when it
  references nothing external; the skill teaches inlining CSS/JS/data so pages stay
  interactive inside the sandbox.
- **Error handling + response etiquette** — react to quota/size/tier/auth errors, and
  surface the exact returned URL (never invent one) — the URL is the deliverable.

Surfaces it can drive: the **`sharedrop` CLI** (primary); the hosted **MCP server**
(`whoami`, `create_upload`/`finalize_upload`/`finalize_bundle`, `upload_html`,
`upload_file`, `paste_html`, `get_page`, `fetch_page`, `list_pages`, `update_page`,
`delete_page`, `share_with_email`, `share_page`, `list_shares`, `revoke_share`,
`create_ephemeral_link`, `list_ephemeral_links`, `revoke_ephemeral_link`); and the
**REST API**.

Supported upload types: HTML, MHTML web archives, Markdown, PDF, and images
(PNG, JPEG, WebP, GIF, AVIF, BMP, ICO, APNG, SVG, HEIC/HEIF, TIFF).

## Install the skill

### Agents with a skills directory (one command)

Claude Code, OpenAI Codex, OpenClaw, Hermes, Cursor, OpenCode, and ~65 other agents via
the [skills CLI](https://github.com/vercel-labs/skills) — it auto-detects every agent on
your machine and installs globally for each:

```bash
npx skills add ShareDropCloud/skills --skill sharedrop -g
```

No npm? The fallback installer drops `SKILL.md` straight into each agent's global skills
directory:

```bash
curl -fsSL https://sharedrop.cloud/skill.sh | bash
```

Per-project instead of global: drop the `-g` flag, or fetch the file directly:

```bash
mkdir -p .claude/skills/sharedrop
curl -fsSL https://sharedrop.cloud/sharedrop-skill.md -o .claude/skills/sharedrop/SKILL.md
```

### Claude Desktop / claude.ai

Skills are uploaded, not installed on disk: download
[sharedrop-skill.zip](https://sharedrop.cloud/sharedrop-skill.zip) and add it under
**Settings → Capabilities → Skills**.

### ChatGPT

No skill mechanism — use the MCP connector plus project instructions instead:
[sharedrop.cloud/docs/install-skill](https://sharedrop.cloud/docs/install-skill).

## Using Sharedrop (what the skill drives)

### The `sharedrop` CLI — recommended

The fastest, most reliable surface for an agent: one command per job, authenticate once,
JSON output with `--json`.

```bash
npm install -g @sharedrop/cli                 # the `sharedrop` binary (or: npx @sharedrop/cli)
sharedrop login                               # browser auth — or set SHAREDROP_TOKEN=sd_... (CI/headless)
sharedrop upload report.html --title "Q4 Report" --json    # → a stable sharedrop.cloud URL
sharedrop update <id> report.html             # revise in place — same URL, new version
sharedrop fetch <id>                          # read a page's raw content back into context
```

Full reference: [sharedrop.cloud/docs/cli](https://sharedrop.cloud/docs/cli).

### MCP server — fallback for clients with no shell

Remote-only HTTP; OAuth on first connect, or a Bearer API key from
[dashboard → settings → API keys](https://sharedrop.cloud/dashboard/settings/api-keys):

```json
{
  "mcpServers": {
    "sharedrop": {
      "type": "http",
      "url": "https://sharedrop.cloud/api/mcp"
    }
  }
}
```

Per-client setup (Claude Desktop, Claude Code, Cursor, Windsurf, Codex):
[dashboard → settings → MCP](https://sharedrop.cloud/dashboard/settings/mcp).

### REST API — last resort

Streamed `sign → PUT → finalize` upload for any file type:
[sharedrop.cloud/docs/api-reference](https://sharedrop.cloud/docs/api-reference).

## Verify it works

Ask your agent: *"share a hello-world page with me"* — it should upload via the CLI and
reply with a `sharedrop.cloud` URL on its own line.

## About this repo

This is a **read-only mirror** for the skills CLI. The canonical skill source lives in
the Sharedrop app repo and is published at
[sharedrop.cloud/sharedrop-skill.md](https://sharedrop.cloud/sharedrop-skill.md); both the
skill and this README are regenerated here by an automated sync, so they always match the
live docs. Issues and feedback: hello@sharedrop.cloud.

Skills run with your agent's full permissions — read
[skills/sharedrop/SKILL.md](skills/sharedrop/SKILL.md) before installing, as you should
for any skill.

Licensed under [MIT](LICENSE).
