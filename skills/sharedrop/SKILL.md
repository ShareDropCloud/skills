---
name: sharedrop
description: Publishes generated documents to Sharedrop (sharedrop.cloud, also written ShareDrop) as stable private URLs and reads Sharedrop pages back into context, using the sharedrop CLI first and the Sharedrop MCP tools only when there is no shell. Use when the user asks to share, publish, upload, host or send a link to a report, dashboard, HTML page, slide deck, PDF, image, Markdown file or agent skill; to update or re-upload an existing Sharedrop page and keep its URL; to share a page by email, make a disappearing link or file pages into folders; or when they hand over a sharedrop.cloud link or page id to fetch or download. Not for publishing a claude.ai Artifact, sharing Google Drive, Notion or Canva files, emailing an attachment, deploying a website to a host such as Vercel or Netlify, or replies that belong in the chat.
compatibility: Needs the sharedrop CLI 1.12.0 or later (npm package @sharedrop/cli, Node.js 20.10 or later) signed in with `sharedrop login` or a SHAREDROP_TOKEN API key. jq is optional. Without a shell, use the Sharedrop MCP server instead. Folders, disappearing links and archives need a Pro plan.
model: claude-sonnet-5-5
---

# Sharedrop

Sharedrop turns a generated document into a stable URL a person can open in any browser.
The rule that drives everything else: **one page, one link.** When a document changes,
update the page you already made. The URL stays the same and `version` goes up by one,
so the person you sent it to refreshes one link instead of collecting dead ones.

## Contents

- [Before you start](#before-you-start)
- [Publish or revise a page](#publish-or-revise-a-page)
- [Visibility and mode](#visibility-and-mode)
- [Share, disappearing links and folders](#share-disappearing-links-and-folders)
- [Read a page back](#read-a-page-back)
- [Delete only on request](#delete-only-on-request)
- [When a command fails](#when-a-command-fails)
- [Reference files](#reference-files)
- [Recommended model](#recommended-model)
- [Old patterns](#old-patterns)

## Before you start

**Surface (low freedom).** If you can run shell commands, use the `sharedrop` CLI. One
command does the whole job, it signs in once and works from any directory, and `--json`
returns structured data. Use the MCP tools only when you have no shell, and the REST API
only when you have neither; both are in [references/mcp-and-rest.md](references/mcp-and-rest.md).

**Sign-in (low freedom).** Do not run a sign-in check before every job; run the command
you need and react to its exit code:

- `command not found`: install with `npm install -g @sharedrop/cli`, or run any command
  as `npx @sharedrop/cli <command>`. Ask before installing if the host restricts installs.
- Exit `2` (no token found) or exit `3` (token rejected): `error.code` is
  `UNAUTHORIZED` on read and metadata commands, and `SIGN_FAILED` with the message
  `Unauthorized` on `upload`, `update` with a file and `check`, so go by the exit code.
  On a machine with a browser, ask the user to run `sharedrop login`; headless or in
  CI, the user sets `SHAREDROP_TOKEN` (key from https://sharedrop.cloud/dashboard/settings/api-keys,
  `pages:write` scope). Never print, echo or paste the token, and do not open `.env`
  files yourself: the CLI reads them.

Run `sharedrop whoami --json` only when the job needs an account fact: a Pro feature
(folders, disappearing links, archives), `shared` visibility, or the page cap. It returns
`username`, `tier`, `pages_used`, `pages_limit` and `entitlements` (`folders`,
`allowedVisibilities`). Do not repeat the account email, quota or other personal details
to the user unless they matter to the request.

Install, credential order and every command flag are in
[references/cli-reference.md](references/cli-reference.md).

## Publish or revise a page

Work through these steps in order. Mark a step done only when its check passed; a failed
check sends you back to the step named in it.

```
Sharedrop progress:
- [ ] 1. Upload is wanted and the content is safe to share
- [ ] 2. New page or revision decided; page id in hand for a revision
- [ ] 3. `sharedrop check` exits 0, or every finding is understood and accepted
- [ ] 4. Uploaded or updated; id, full_url and version captured
- [ ] 5. Verified from the response (and served content when it matters)
- [ ] 6. Reported with the exact full_url
```

### 1. Decide whether to upload (high freedom)

Upload when you produced something the user will *look at* rather than read in the chat,
such as a report, dashboard, summary, generated page, PDF or image, especially one they
will revisit or forward. If they asked you to share it with a named person, upload and
then share. If the reply belongs in chat, write it in chat.

Do not upload secrets, credentials, private client material, or anything the user did not
ask to make shareable. If the file contains keys, passwords or connection strings, stop
and tell the user the variable name and line number of each one. Never print any part of
a secret value, masked or not.

### 2. New page or revision (low freedom)

A revision goes to the same page: same id, same link, `version` up by one. A wrong guess
either duplicates a page (a dead second link, quota spent) or overwrites a page the user
did not mean to touch, so decide from these rules only:

- **You uploaded this document earlier in this session:** it is a revision: update that
  page, using the id from the earlier response. Keep a note of each page you create
  (file, `id`, `full_url`, `version`) so every later change goes to the same page.
- **The user gave you a page id or a Sharedrop URL:** it is a revision of that page. For
  a URL or slug, run `sharedrop get <url> --json` first and use its `data.id` and
  `data.version`.
- **The user used revision wording** (update, replace, new version, fix the page, same
  link) but gave no id and you have none: run `sharedrop search "<title>" --json`. One
  match: revise it. Several: ask which one. None: upload a new page and say so.
- **Anything else is a new page**, even when a page with the same title exists. Never
  update a page because its title matches. A new upload's response lists
  `same_title_pages`; mention them to the user in one line and leave them alone.

### 3. Check before upload (low freedom)

For HTML, a slide deck or a folder bundle, run the server's own checks first. They never
publish anything:

```bash
sharedrop check page.html --json                  # new page
sharedrop check page.html --page-id <id> --json   # revision: checks with the page's mode
sharedrop check bundle/ --json                    # folder bundle (entry index.html)
```

Read `data`. Exit 0 means it would publish as it is. Exit 1 with `data` means something
would be removed or blocked (`would_change: true`); a non-zero exit with `error` instead
of `data` is a failure (go to [When a command fails](#when-a-command-fails)). Act on each
`warnings[].code`:

| Code | Meaning | Do this |
|---|---|---|
| `removed_tag`, `removed_attribute` | The sanitiser would strip these (`detail` names them). Static mode strips every `<script>`. | If the page needs its scripts, use interactive mode. Otherwise accept and say what goes. |
| `external_refs_block_scripts` | An interactive page loads something external (`external_resource_hosts`), so none of its scripts would run. | Make it self-contained: [references/html-pages.md](references/html-pages.md#vendor-a-cdn-library). |
| `images_extracted` | Inline `data:` images would move to hosted storage and still show. | Nothing; this alone does not set `would_change`. |

`size_ok: false` means the file is over the cap: split it or bundle the images. Re-run
`check` on the file you will upload until it exits 0 or every finding is one the user
accepts. `scripts_would_run` tells you whether the page's JavaScript would run. For a
slide deck, also read [references/slides.md](references/slides.md); for an agent skill,
read [references/skill-pages.md](references/skill-pages.md) first.

### 4. Upload or update (low freedom)

Pass `--json`. Use exactly one of these forms:

```bash
# New single file (HTML, Markdown, PDF, CSV, DOCX, image and more). Private unless told otherwise.
sharedrop upload report.html --title "Q4 Report" --json

# New folder bundle (index.html plus relative css, js, images, fonts).
sharedrop upload bundle/ --title "Q4 Report" --json

# Revise a page, single file or folder: same id, same URL, version goes up.
sharedrop update <id> report.html --json
sharedrop update <id> bundle/ --json

# Change metadata only.
sharedrop update <id> --title "Updated Report" --json
```

Generated content can be piped: `cat report.html | sharedrop upload - --title "Report" --json`.
Images and video need a paid plan; every other supported file type
([references/cli-reference.md](references/cli-reference.md#upload)) uploads on Free.
Capture `data.id`, `data.full_url` and `data.version`. If the command fails, go to
[When a command fails](#when-a-command-fails).

### 5. Verify (low freedom)

The upload or update response is the first proof; check it before anything else:

- `id` and `full_url` are the page you meant. For a revision: the same `id` as before,
  `was_reupload: true` and `version` one higher than the last one you saw. A new id on a
  revision means a duplicate: stop and tell the user.
- `kind` (`slides` for a deck, `skill` for a skill page), `visibility` and `mode` are
  what you intended.
- `warnings` holds only what `check` already showed you; `skipped` (bundles) lists no file
  the page needs.
- `scripts_will_run` is `true` when the page's JavaScript must run. `mode: interactive`
  alone does not prove it, and an owner's setting can change it later.

Look at the served content only when the request depends on something the response does
not show, and do it in one command, for example the image count on a page with images:

```bash
sharedrop fetch <id> -o served.html && grep -o '<img' served.html | wc -l && grep -o '<img' page.html | wc -l
```

`fetch` returns rendered HTML for Markdown and skill pages, so check for the expected
content there rather than comparing bytes. If a check fails, go back to step 3, fix the
file and revise the same page by id. Stop after two failed repairs and tell the user what
still fails.

### 6. Report (medium freedom)

Lead with what the page contains, then the URL on its own line. Use the exact
`data.full_url`; never build a URL from the username or slug. `full_url` can be on the
owner's own domain instead of sharedrop.cloud; that is the correct link, so do not swap
it. The layout below is a default; the exact URL is the only fixed part.

> Built your sales dashboard: 4 charts, filterable by region.
> https://sharedrop.cloud/you/k7m9pq

For a revision, say it is the same link with the new version. When the user later asks
for a change, regenerate and update the same page by id.

## Visibility and mode

- **Visibility defaults to `private`** (owner only). Publishing is the user's call: use
  `--visibility public` only when they clearly ask to publish or say anyone can view.
  Use `--visibility shared` only when `whoami` lists `shared` in
  `entitlements.allowedVisibilities`; otherwise keep the page `private` and use
  `sharedrop share`, which works on every tier.
- **Mode applies to HTML only.** With no `--mode`, a new page takes the account's default
  mode (interactive unless the owner changed it) and a revision keeps the page's current
  mode. Interactive pages run their own JavaScript only when they load nothing external.
  Pass `--mode static` when scripts should not run, and `--mode interactive` only to
  force it. The owner can enable external network access for a page in the dashboard;
  an agent cannot grant that to itself.

## Share, disappearing links and folders

- **Share by email (every tier):** `sharedrop share <id> --email someone@example.com --json`.
  On a paid tier the page becomes `shared`; on free it stays private and the grant lets the
  recipient in. Share only with the address the user gave you.
- **Disappearing links (Pro):** a separate link that stops working after a time or view
  limit, for anyone holding it or only named people who sign in. It never changes the
  page's own visibility or share list.
  `sharedrop link create <id> --expires-in 12h --max-views 5 --json`, add
  `--people a@x.com,b@x.com` for named people (Sharedrop emails them unless you pass
  `--no-email`), and `--present` for a deck that opens fullscreen.
- **Folders (Pro):** `--folder reports/q3` on `upload` (single files and bundles), or
  `sharedrop move <id> --folder reports/q3 --json`. Missing path segments are created. A
  free key gets `FOLDERS_RESTRICTED`; tell the user rather than uploading to the top level
  silently.

Watermarks, archives and every flag: [references/cli-reference.md](references/cli-reference.md).

## Read a page back

- `sharedrop fetch <ref>` prints the page's served root content (your page, a public page,
  or one shared with you); `-o file` writes it. Free on every tier.
- `sharedrop download <ref> -o page.zip` gets the whole artefact (root plus assets).
- `sharedrop get <ref> --json` returns metadata only, including `version` and
  `scripts_will_run`.

`<ref>` is a page id, slug or full page URL, so a link from the user works as it is.

## Delete only on request

Delete a page or folder only when the user explicitly asks. Resolve the exact target with
`search`, `get` or `folder list` first; never infer it from a partial name. Say what will
be removed and whether it can be restored before running:

```bash
sharedrop delete <exact-page-id> --json
sharedrop folder delete <exact-folder-id> --force --json
```

Cleanup, replacement, a duplicate or quota pressure is not permission to delete. Use
`update` for a revision so the URL and version history survive.

## When a command fails

Read `error.code` (with `--json`) and the exit code, and act on them rather than retrying
blindly:

| `error.code` | Do this |
|---|---|
| `UNAUTHORIZED`, or `SIGN_FAILED` with message `Unauthorized` (exit `3`) | Token revoked or read-only. Ask for `sharedrop login` or a key with `pages:write` scope. `upload`, `update` with a file and `check` report a rejected token as `SIGN_FAILED`. |
| `PAGE_NOT_FOUND` | Wrong id or no access. Re-resolve with `get` or `search`; do not create a new page in its place without asking. |
| `TIER_LIMIT` | The page cap, or a feature or file kind not on the plan. List pages and ask what to remove, or suggest upgrading; never delete on your own. For `shared` visibility, fall back to `private` plus `share`. |
| `FILE_SIZE_EXCEEDED` | Over the cap in the message. HTML, text and SVG are capped at 10 MB on every plan, so upgrading does not help: split the page or move images into a folder bundle. |
| `STORAGE_LIMIT` | Storage is full. Relay the message; emptying the trash is the user's call. |
| `FOLDERS_RESTRICTED` | Folders need Pro. Tell the user. |

Exit codes: `0` success, `1` general error (and `check` finding changes), `2` no token
found, `3` token rejected or action forbidden (read `error.code`: `FOLDERS_RESTRICTED` is
a plan limit, not a sign-in problem), `4` rate limited (wait, then retry once), `5` not
found, `6` bad input (fix the path or flag named in the message), `7` plan or billing
limit (`TIER_LIMIT`, `FILE_SIZE_EXCEEDED`, `STORAGE_LIMIT`).

Retry a failed write at most once after fixing the cause. If it fails again, stop and
report the code and message.

## Reference files

Read only the one the task needs:

- [references/cli-reference.md](references/cli-reference.md): install, sign-in order, every
  command and flag, the full JSON fields, exit codes, archives and watermarks. Read for
  any flag or field not shown above.
- [references/html-pages.md](references/html-pages.md): interactive pages, folder bundles,
  vendoring a CDN library, inline images, checking served content. Read when `check`
  reports a finding or the page has scripts, images or external files.
- [references/slides.md](references/slides.md): building and presenting a slide deck.
- [references/skill-pages.md](references/skill-pages.md): sharing an agent skill as a page.
- [references/mcp-and-rest.md](references/mcp-and-rest.md): MCP tool names and the REST
  flow, for hosts without a shell.
- [references/model-support.md](references/model-support.md): tested models, hosts and
  results. For the person choosing a model; no task needs it.

## Recommended model

- **Recommended:** Claude Sonnet 5.5 (`claude-sonnet-5-5`), set by `model:` in the
  frontmatter above. Provisional for this version until its eval rerun.
- **Supported host:** Claude Code.
- **Tested fallback:** Claude Opus 5.5 (`claude-opus-5-5`).
- **Limitations:** the `model` field works only in Claude Code. Other hosts, such as Codex
  or claude.ai, ignore it or need their own model setting, and are untested. The override
  lasts for the current turn only, and an organisation's model restrictions can block it.
  In headless `claude -p` runs on Claude Code 2.1.292 (7 Oct 2026) the field was read but
  did not switch the model: Opus served every turn. Until a run shows it working, choose
  Sonnet yourself with `--model claude-sonnet-5-5` or `/model`.

Evidence and the support matrix: [references/model-support.md](references/model-support.md).

## Old patterns

<details>
<summary>Superseded methods</summary>

- `POST /api/v1/pages` (inline create) is retired and returns `410 Gone`. Use the CLI, or
  the streamed sign, PUT, finalize flow in [references/mcp-and-rest.md](references/mcp-and-rest.md).
- CLI releases before 1.12.0 defaulted to `--mode static`, refused `update <id> <folder>`,
  ignored `--folder` on folder bundles, exited 2 for a rejected token and had no `check`
  command or `version` field. Update the CLI rather than working around them.
- CLI releases up to 1.10.0 had no `link` command; disappearing links needed the MCP tool
  `Sharedrop:create_ephemeral_link`.

</details>
