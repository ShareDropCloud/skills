---
name: sharedrop
description: |
  Share AI-generated documents through Sharedrop and read shared pages back into context. Use when the user asks to share, publish, save, or update a report, dashboard, document, slide deck, PDF, image, or other generated artefact; requests a stable URL; asks to share a page by email; or provides a Sharedrop page to fetch. Also use for HTML slide decks intended for presentation. Prefer the authenticated `sharedrop` CLI, default to private visibility, update existing pages by ID to preserve their URLs, and surface the exact URL returned by Sharedrop.
---

# Sharedrop

Sharedrop turns a document you generated into a stable URL a human can open in any browser.
The mental model that makes everything else fall into place: **one page, one URL, forever.**
When you regenerate that report, you re-upload to the *same* page. The URL doesn't change
and a version is recorded, so the person you sent it to keeps refreshing one link instead
of collecting a pile of dead ones.

## Reach for the CLI first

If you can run shell commands, use the `sharedrop` CLI. It's the surface built for agents:
one command does the whole job, it authenticates once and then works from any directory,
and with `--json` every response is structured data you can parse. You don't have to wire up
an MCP server or hand-roll a multi-step signed-upload dance, because the CLI does that
for you. The MCP and REST paths near the bottom exist only for agents that *can't* open a
shell.

### Install

```bash
npm install -g @sharedrop/cli      # installs the `sharedrop` binary
```

No global install? `npx @sharedrop/cli upload report.html` runs the same thing on demand.

### Authenticate once

The CLI takes the first credential it finds, in this order: a `--token` flag, then the
`SHAREDROP_TOKEN` environment variable, then a `.env` in the working directory, then a key
saved by `sharedrop login`. (It targets `https://sharedrop.cloud` unless you pass `--url` or
set `SHAREDROP_URL`.) Pick the one that fits where you're running:

- **On a machine with a browser**: run `sharedrop login` once. It opens the browser, mints
  a CLI key, and stores it in your OS config directory, so every later command just works
  with nothing to copy-paste.
- **Headless, in CI, or in a sandbox with no terminal**: set `SHAREDROP_TOKEN=sd_…`
  (create a key at https://sharedrop.cloud/dashboard/settings/api-keys). `login` needs an
  interactive terminal; the env var doesn't, which is why it's the right choice for agents.

Run `sharedrop whoami` to confirm. It reports the username, plan tier, and remaining quota,
which is also how you learn what the account is allowed to do before you try something.
Use that output for the preflight decision; do not repeat the account email, quota, or other
personal metadata to the user unless it is relevant to the request.

### The commands you'll use

Pass `--json` so you get `{ "data": … }` on success and `{ "error": { "code", "message" } }`
on failure (piped, non-interactive shells default to JSON anyway). The upload and update
responses carry the live URL in `data.full_url`. Surface *that exact value* to the user. It
is the deliverable; never reconstruct or guess a URL.

```bash
# UPLOAD: from a file, or pipe generated content over stdin with `-`.
sharedrop upload report.html --title "Q4 Report" --visibility private --json
cat report.html | sharedrop upload - --title "Generated Report" --json

# Lift the URL straight out of the response:
URL=$(cat report.html | sharedrop upload - --json | jq -r '.data.full_url')

# UPDATE an existing page: same URL, new version recorded. Pass a file to replace the
# content, or only flags to change the title/visibility. Re-uploading without the page id
# instead mints a *duplicate* page and burns quota, so to revise, always update by id.
sharedrop update <id> report.html --json
sharedrop update <id> --title "Updated Report" --visibility public --json

# READ a page's served root content back into context: your own, a public one, or one
# shared to you. `get` returns only metadata. Free on every tier.
sharedrop fetch <id>                 # raw bytes to stdout, pipe into another tool
sharedrop fetch <id> -o report.html  # …or write a file

# FIND / INSPECT / MANAGE
sharedrop list --json                       # your pages (each row carries the id)
sharedrop search "q4 report" --json         # matches title, slug, id, and file type at once
sharedrop get <ref> --json                  # one page's metadata
sharedrop share <id> --email alice@example.com --json
sharedrop download <id> -o page.zip --json  # the full artefact (root + assets) as a zip
```

Refer to a page by the **id** from `list` or the upload response. `fetch` and `download`
need that id specifically. `get`, `update`, `delete`, and `share` are more forgiving: a
slug or a full page URL works there too, so you can paste whatever the user handed you.

`fetch` is not byte-preserving for every kind. Rendered kinds such as Markdown and native
skill pages can return sanitised or rendered HTML rather than the original source file.
Use checksums only for kinds documented as byte-preserving. For rendered kinds, verify the
metadata with `get` and inspect the fetched page for the expected semantic content.

Control flow can lean on exit codes instead of scraping text: `0` success, `1` general
error, `2` no token, `3` token rejected, `4` rate limited, `5` not found, `6` bad input.

### Organise into folders (Pro plan or higher)

Pro accounts can file pages into nested folders. `--folder` takes a folder id or a slash
path like `reports/2026/q3` (missing segments are auto-created); a free key gets a
`FOLDERS_RESTRICTED` error with an upgrade link instead of a silent root fallback.

```bash
sharedrop upload report.html --folder reports/q3 --json   # file a new page as it lands
sharedrop move <id> --folder reports/q3 --json            # move an existing page in
sharedrop move <id> --root --json                         # ...or back to the top level
sharedrop list --folder reports/q3 --json                 # list a folder's pages

sharedrop folder create reports/2026/q3 --json            # nested; auto-creates segments
sharedrop folder list --json                              # top-level folders (--parent <id> for children)
sharedrop folder rename <id> "Q3 Reports" --json
sharedrop folder move <id> --root --json                  # reparent (--parent <id>) or --root
sharedrop folder restore <id> --json                      # undo within 30 days
```

### Share an agent skill

A standard agent skill can be a directory containing `SKILL.md` plus `references/`,
`scripts/`, `assets/`, or agent metadata. A Sharedrop native skill page is one uploaded
file. Inspect the source skill tree before uploading:

- If `SKILL.md` is genuinely self-contained, upload it directly. Sharedrop recognises
  `SKILL.md`, `*.skill.md`, and `*.skill` as native skill pages.
- If the skill depends on other files, do not upload `SKILL.md` alone. Create a
  self-contained distribution copy such as `<name>.skill` by embedding the required
  reference material and replacing local relative links. Keep the original skill
  directory unchanged as the maintainable source.
- Do not imply that a viewer URL is an installable multi-file skill package. If the
  recipient needs the original directory, provide it through a distribution method that
  preserves the complete tree.

Before creating a page, search for the title. Update the existing page by id when it is a
revision; create a new page only when no matching page exists.

```bash
sharedrop whoami --json
sharedrop folder list --json
sharedrop search "Ascend Notion CRM" --json

# New self-contained skill:
sharedrop upload ascend-notion-crm.skill \
  --title "Ascend Notion CRM" \
  --visibility private \
  --folder Skills \
  --json

# Revision: preserve the established URL.
sharedrop update <id> ascend-notion-crm.skill --json
```

Verify a skill upload before reporting success:

```bash
sharedrop get <id> --json
sharedrop list --folder Skills --json
sharedrop fetch <id> -o rendered-skill.html
```

Confirm that `get` reports `kind: "skill"`, the intended visibility, and the exact
`full_url`; confirm the page id appears in the intended folder; then inspect the rendered
page for the expected workflow and embedded reference content. Do not require its checksum
to match the Markdown source.

### A typical run

**Input:** the user says *"make me a sales dashboard and send me the link."*

```bash
# you generated dashboard.html, then:
URL=$(sharedrop upload dashboard.html --title "Sales Dashboard" --json | jq -r '.data.full_url')
```

**Output:** lead with what's in it, end with the link on its own line:

> Built your sales dashboard: 4 charts, filterable by region.
> https://sharedrop.cloud/you/k7m9pq

Later they say *"update it with March numbers"*, so you regenerate and
`sharedrop update k7m9pq dashboard.html`; the link they already have now shows March.

Full CLI reference: https://sharedrop.cloud/docs/cli

## Verify every write

An upload or update response is not sufficient proof by itself:

1. Capture the returned page id and exact `data.full_url`.
2. Run `sharedrop get <id> --json` and confirm title, kind, visibility, and updated time.
3. If a folder was requested, run `sharedrop list --folder <id-or-path> --json` and confirm
   the page id is present.
4. Verify content in a kind-appropriate way: byte comparison only for byte-preserving
   kinds, semantic inspection for rendered kinds.
5. Report the exact returned URL, not one reconstructed from the username or slug.

## When to use it (and when not)

Upload whenever you've produced something the user will *look at* rather than read in the
chat stream, whether a report, dashboard, summary, generated page, PDF, or image,
especially if they'll want to revisit or forward it. If they asked you to share it with a
named person, upload and then `share`. If they handed you a Sharedrop page and need its
contents, `fetch`.

Don't upload secrets, credentials, private client material, or anything the user didn't ask
to make shareable. Do not use Sharedrop to dodge writing a normal answer. If the reply
belongs in chat, just write it.

## Choosing visibility and mode

- **Visibility defaults to `private`** (owner only) for good reason: publishing is the
  user's call, not yours. Use `--visibility public` only when they clearly ask to publish or
  say "anyone can view". `shared` (named email grants on the page) needs a paid tier; on a
  free account, keep the page `private` and use `sharedrop share`, which grants a specific
  person access and works on every tier.
- **Mode applies to HTML only** and defaults to `static` (scripts get stripped, which is
  safe and fine for most reports). Pass `--mode interactive` only when the page genuinely
  needs to run JavaScript, and only when it is fully self-contained (see below).

## Interactive pages must be fully self-contained

Interactive pages run in a locked-down, offline sandbox. If one references *anything*
external (a CDN script, a Google Font, a remote image, a tracking pixel, an external API,
or a `<base href>`), Sharedrop can't trust it and disables **all** of its JavaScript,
serving it as static with a banner. So build interactive pages closed: inline the CSS in
`<style>`, the JS in `<script>`, and small images as `data:` URIs; never `fetch()` at
runtime. Drive tabs, filters, and charts from data you've already inlined. For heavier
assets, upload a multi-file bundle and reference them by relative path. A page that truly
needs the open internet only works if the human owner enables external-network mode for it
in the dashboard. An agent can't grant that to itself.

## Build a slide deck and present it

Sharedrop presents HTML decks fullscreen, so "make me a presentation" ends at a URL rather
than a PowerPoint export. Decks work on **every plan**, including Free.

There is no CLI flag for this and none is needed: put the marker in the HTML and any upload
path (CLI, drag-and-drop, API, MCP) produces a deck.

```html
<meta name="sharedrop:kind" content="slides" />
```

Structure the body as **one top-level `<section>` per slide**. Present shows one at a time
and steps through them on arrow key, space, click, or tap:

```html
<body>
  <section><h1>Q3 review</h1></section>
  <section>
    <h2>Three shifts</h2>
    <ul>
      <li data-sd-fragment>Pilots became practice</li>
      <li data-sd-fragment>Hours returned to the business</li>
    </ul>
  </section>
</body>
```

Two hooks make a deck feel like a presenter tool, and both are plain CSS you write yourself:

- **`data-sd-fragment`** on any element inside a slide reveals it one step at a time, the
  way a bullet list builds. The next step only moves to the next slide once every fragment
  is shown. They fade in by default; style the reveal yourself off `__sd-frag-visible`.
- **`__sd-active`** is the class Present adds to the slide currently on screen. Key an
  entrance animation to it and the slide animates in as it lands.

Present never restyles your deck. Your CSS is exactly what the audience sees.

```bash
# deck.html carries the marker, so this is just a normal upload
URL=$(sharedrop upload deck.html --title "Q3 review" --json | jq -r '.data.full_url')
sharedrop get <id> --json | jq -r '.data.kind'    # -> slides, check before claiming success

# present it: add ?present=1 to the page URL
echo "$URL?present=1"
```

**Static or interactive?** A static deck still animates. CSS transitions, fragments, and
slide-entry classes all work, because Present supplies the navigation. Upload
`--mode interactive` only when the deck's *own* JavaScript must run (count-ups, charts,
canvas); Present keeps driving the slides either way. The self-contained rule above still
applies, so inline everything.

For an unattended screen, add `autoplay=<whole seconds>` to advance and loop forever:
`…?present=1&autoplay=15`.

**On MCP or REST instead of a shell:** pass `slides: true` to **`finalize_upload`** (it is
ignored on `sign`, and on non-HTML files). To hand someone a link that opens straight into
fullscreen on a TV, Pro accounts can create a present-only disappearing link with
`create_ephemeral_link({ page_id, expires_in_seconds, present_only: true })`.

Full guide: https://sharedrop.cloud/docs/slides

## Sharing, expiring links, and watermarks

`sharedrop share <id> --email someone@example.com` grants one person access; on a paid tier
the page auto-promotes to `shared` visibility, and on free tier it stays private but the
recipient can still open it through the grant. Two Pro-only extras aren't in the CLI:
**disappearing links** that expire by time or view count, and a **watermark overlay**.
Reach for the MCP tools (`create_ephemeral_link`, `update_page` with `watermark_enabled`)
or the dashboard for those.

## Destructive actions

Delete a page or folder only when the user explicitly asks. Resolve the exact target with
`search`, `get`, or `folder list` first; never infer a destructive target from a partial
name. State what will be removed and whether it is recoverable before running:

```bash
sharedrop delete <exact-page-id> --json
sharedrop folder delete <exact-folder-id> --force --json
```

Do not treat cleanup, replacement, or quota pressure as implicit permission to delete.
Use `update` for a revision so the existing URL and version history are preserved.

## When something goes wrong

The `error.code` in a failed response tells you what to do. React to it rather than
retrying blindly:

- `PAGE_LIMIT_REACHED`: the free-tier page cap. `list`, ask the user what to remove, or
  suggest upgrading.
- `FILE_SIZE_EXCEEDED`: over the tier's size limit (the message gives the cap). Compress
  inline images or split the document.
- `TIER_LIMIT`: a paid-only action on a free plan (e.g. `shared` visibility, an image
  upload). Tell the user it needs an upgrade instead of retrying; for `shared` specifically,
  fall back to `private` + `share`, which works anywhere.
- `UNAUTHORIZED`: the token is missing, revoked, or read-only. Re-run `sharedrop login`, or
  point the user at https://sharedrop.cloud/dashboard/settings/api-keys for a key with
  `pages:write` scope.

## If you can't use the CLI

### MCP server

For MCP-native clients with no shell. It's remote-only HTTP at
`https://sharedrop.cloud/api/mcp`, with OAuth on first connect, or a Bearer `sd_` key:

```json
{
  "mcpServers": {
    "sharedrop": { "type": "http", "url": "https://sharedrop.cloud/api/mcp" }
  }
}
```

Upload without base64 by streaming: `create_upload` → HTTP `PUT` the bytes to the returned
`upload_url` (`Authorization: Bearer <upload_token>`) → `finalize_upload`. Every file,
including HTML, uses the same streamed pipeline; supply `page_id` when replacing an existing
page. Use `fetch_page` to read content back. The rest map onto the CLI verbs:
`whoami`, `get_page`, `list_pages`, `update_page`, `delete_page`, `share_with_email`,
`share_page`, `list_shares`, `revoke_share`, `create_ephemeral_link`,
`list_ephemeral_links`, `revoke_ephemeral_link`, `finalize_bundle`. Pro accounts also get
folder tools (`create_folder`, `list_folders`, `move_page`, `delete_folder`,
`restore_page`), and `finalize_upload` accepts `folder_id` or `folder_path` to file a new
upload straight into a folder.
Setup per client: https://sharedrop.cloud/dashboard/settings/mcp

### REST API

The last resort, with neither a shell nor MCP. Uploading is a streamed three-step flow for
any file type: **sign → PUT the bytes → finalize**. (The old inline `POST /api/v1/pages`
create is retired and now returns `410 Gone`; don't reach for it.)

```bash
# 1. sign: reserve a key and mint a 5-minute upload token
SIGN=$(curl -s -X POST https://sharedrop.cloud/api/upload/sign \
  -H "Authorization: Bearer sd_YOUR_KEY" -H "Content-Type: application/json" \
  -d '{"filename":"report.html","content_type":"text/html","size_bytes":'"$(wc -c <report.html)"'}')
UPLOAD_URL=$(echo "$SIGN" | jq -r '.data.upload_url')
UPLOAD_TOKEN=$(echo "$SIGN" | jq -r '.data.upload_token')
OBJECT_KEY=$(echo "$SIGN" | jq -r '.data.object_key')

# 2. PUT the raw bytes straight to storage (no request-body size cap)
curl -X PUT "$UPLOAD_URL" \
  -H "Authorization: Bearer $UPLOAD_TOKEN" -H "Content-Type: text/html" \
  --data-binary @report.html

# 3. finalize: sanitise + publish; add "page_id":"<id>" to update an existing page
curl -X POST https://sharedrop.cloud/api/upload/finalize \
  -H "Authorization: Bearer sd_YOUR_KEY" -H "Content-Type: application/json" \
  -d '{"object_key":"'"$OBJECT_KEY"'","upload_token":"'"$UPLOAD_TOKEN"'","title":"Q4 Report","visibility":"private"}'

# Read a page's raw content (two-step token handoff)
FETCH_URL=$(curl -s -H "Authorization: Bearer sd_YOUR_KEY" \
  https://sharedrop.cloud/api/v1/pages/<page_id>/fetch | jq -r '.data.fetch_url')
curl -s "$FETCH_URL" -o page.html      # no auth header, the URL token is the credential
```

Full API reference: https://sharedrop.cloud/docs/api-reference
