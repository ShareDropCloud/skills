# Sharedrop Agent Skills

Official agent skills for [**Sharedrop**](https://sharedrop.cloud) — the agent-native
file drop.

```
npx skills add ShareDropCloud/skills --skill sharedrop -g
```

## What is Sharedrop?

Sharedrop is a sharing platform built for a world where AI agents generate most of the
documents humans read. An agent uploads HTML, Markdown, a PDF, or an image; the human
gets a clean, stable URL — viewable in any browser, no sign-up required to read what
you've been granted. Think of it as a shared drop zone between agents and people:

- **Agents upload** via a hosted MCP server, REST API, or CLI — zero friction, no base64,
  streamed uploads for any file type.
- **Humans read** at a stable URL. Re-uploads with the same `page_id` keep the URL
  identical and record a version, so "update that report" never breaks anyone's link.
- **The right people get access.** Pages default to private; share with specific people
  by email, publish publicly on explicit request, or (Pro) mint disappearing links that
  expire by time or view count.
- **Untrusted content stays contained.** Uploaded HTML is sanitised and rendered in a
  locked-down sandbox on a separate origin with strict CSP — interactive pages run
  JavaScript only when fully self-contained.

More: [docs](https://sharedrop.cloud/docs) · [llms.txt](https://sharedrop.cloud/llms.txt)
· [pricing](https://sharedrop.cloud/pricing)

## What is an agent skill?

A skill is a markdown instruction file (`SKILL.md`) that teaches an AI agent *when* and
*how* to use a capability — judgment, not plumbing. The plumbing is Sharedrop's hosted
MCP server (`https://sharedrop.cloud/api/mcp`); the skill makes the agent use it well.

## Skills in this repo

### `sharedrop` — [skills/sharedrop/SKILL.md](skills/sharedrop/SKILL.md)

Teaches an agent to share generated documents through Sharedrop instead of pasting walls
of HTML into chat or attaching files. Specifically:

- **When to upload** — any time it produces a report, dashboard, summary, or other
  rendered artifact the user should see, archive, link to, or hand to someone else; and
  when *not* to (secrets, things the user didn't ask to publish).
- **How to upload** — the streamed flow (`create_upload` → HTTP `PUT` → `finalize_upload`)
  for any file type with no base64; the `upload_html` shortcut for raw HTML.
- **Stable URLs** — re-upload with the same `page_id` so iterative work keeps one link
  and accumulates version history, instead of minting duplicate pages.
- **Sensible visibility** — default `private`; `public` only on explicit request;
  share with specific people via `share_with_email`; disappearing links on Pro.
- **Self-contained interactive pages** — interactive HTML runs scripts only when it
  references nothing external; the skill teaches inlining CSS/JS/data so pages stay
  interactive inside the sandbox.
- **Error recovery** — what to do on quota, file-size, visibility, and auth errors.
- **Response etiquette** — surface the exact returned URL, never invent one; the URL is
  the deliverable.

Tools it drives (via the MCP server): `whoami`, `create_upload`/`finalize_upload`/
`finalize_bundle`, `upload_html`, `upload_file`, `paste_html`, `get_page`, `list_pages`,
`update_page`, `delete_page`, `share_page`, `share_with_email`, `list_shares`,
`revoke_share`, `create_ephemeral_link`, `list_ephemeral_links`, `revoke_ephemeral_link`.

Supported upload types: HTML, MHTML web archives, Markdown, PDF, and images
(PNG, JPEG, WebP, GIF, AVIF, BMP, ICO, APNG, SVG, HEIC/HEIF, TIFF).

## Install

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

### Connect the MCP server (all agents)

The skill assumes the Sharedrop MCP server is connected — remote-only, OAuth on first
connect (or a Bearer API key from
[dashboard → settings → API keys](https://sharedrop.cloud/dashboard/settings/api-keys)):

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

## Verify it works

Ask your agent: *"share a hello-world page with me"* — it should upload and reply with a
`sharedrop.cloud` URL on its own line.

## About this repo

This is a **read-only mirror** for the skills CLI. The canonical skill source lives in
the Sharedrop app repo and is published at
[sharedrop.cloud/sharedrop-skill.md](https://sharedrop.cloud/sharedrop-skill.md); changes
land here via an automated sync. Issues and feedback: hello@sharedrop.cloud.

Skills run with your agent's full permissions — read
[skills/sharedrop/SKILL.md](skills/sharedrop/SKILL.md) before installing, as you should
for any skill.

Licensed under [MIT](LICENSE).
