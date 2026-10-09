# Sharedrop CLI reference

Every command, flag and JSON field the skill uses, checked against the `@sharedrop/cli`
1.12.0 source. Full upstream reference: https://sharedrop.cloud/docs/cli

## Contents

- [Install](#install)
- [Sign-in and target](#sign-in-and-target)
- [Output and exit codes](#output-and-exit-codes)
- [Page references](#page-references)
- [Check](#check)
- [Upload](#upload)
- [Upload and update JSON](#upload-and-update-json)
- [Update](#update)
- [Read back](#read-back)
- [Find and inspect](#find-and-inspect)
- [Share and disappearing links](#share-and-disappearing-links)
- [Folders and trash](#folders-and-trash)
- [Large files](#large-files)
- [Watermark](#watermark)
- [Delete](#delete)

## Install

```bash
npm install -g @sharedrop/cli      # installs the `sharedrop` binary (needs Node.js 20.10+)
npx @sharedrop/cli <command>       # same thing without a global install
sharedrop --version                # confirm it runs; `check` needs 1.12.0 or later
```

## Sign-in and target

The CLI takes the first credential it finds, in this order:

1. `--token <sd_ key>` flag
2. `SHAREDROP_TOKEN` environment variable
3. a `.env` file in the working directory (the CLI reads it; you do not need to)
4. a key saved by `sharedrop login`

- **Machine with a browser:** `sharedrop login` once. It opens the browser, mints a CLI
  key and stores it in the OS config directory. It needs an interactive terminal, so an
  agent usually asks the user to run it.
- **Headless, CI or sandbox:** set `SHAREDROP_TOKEN=sd_...` from
  https://sharedrop.cloud/dashboard/settings/api-keys. Prefer an env var to `--token`,
  which puts the key in shell history and process lists.

The CLI targets `https://sharedrop.cloud` unless `--url` or `SHAREDROP_URL` says otherwise.

`sharedrop whoami --json` returns `username`, `email`, `tier`, `pages_used`, `pages_limit`
(`-1` means unlimited), `storage_used` and `entitlements` (`folders`,
`allowedVisibilities`, `maxFileSizeBytes`, `maxTextFileSizeBytes`, `maxArchiveBytes`).

## Output and exit codes

With `--json` (the default in piped, non-interactive shells) success is `{ "data": ... }`
and failure is `{ "error": { "code", "message" } }`. A plan or billing refusal puts the
whole billing envelope under `error` (`code`, `message`, `currentTier`, `upgradeUrl`, and
`limitBytes` for a size refusal). `folder list` is the exception on success: it returns
`{ "folders": [ ... ] }` with no `data` wrapper.

```bash
URL=$(sharedrop upload report.html --json | jq -r '.data.full_url')
```

| Exit | Meaning |
|---|---|
| `0` | Success |
| `1` | General error; also `check` when the upload would change or block something |
| `2` | No token found locally. The message is plain text, not JSON |
| `3` | The server rejected the token (401) or refused the action (403, for example `FOLDERS_RESTRICTED` or a read-only key). A rejected token is `error.code` `UNAUTHORIZED` on read and metadata commands and `SIGN_FAILED` with message `Unauthorized` on `upload`, `update` with a file and `check`; rely on the exit code |
| `4` | Rate limited |
| `5` | Not found |
| `6` | Bad input; a local file that does not exist exits `6` before any network call |
| `7` | Plan or billing limit (402): `TIER_LIMIT`, `FILE_SIZE_EXCEEDED`, `STORAGE_LIMIT` |

An expired one-off upload token (`TOKEN_EXPIRED`) exits `1`; run the command again. When
another upload of the same page is still finishing, the CLI waits and retries the final
step by itself; if it gives up, the JSON error carries `retryable` and
`retry_after_seconds`.

## Page references

`get`, `fetch`, `download`, `update`, `delete`, `share` and `link` take a page id, slug or
full page URL. `upload --page-id` takes the id only. To revise a page you know by URL,
run `get <url> --json` and use `data.id`.

## Check

```bash
sharedrop check <file|folder> [options]
```

Streams the file to quarantine, runs the server's real upload checks, deletes the copy and
prints what it found. It never publishes. It needs the same sign-in as `upload`.

| Flag | Meaning |
|---|---|
| `--mode <mode>` | Check as `static` or `interactive` (default: what `upload` would use) |
| `--page-id <id>` | Check as a re-upload of this page, with its mode |
| `--slides` | Check a single HTML file as a slide deck |
| `--entry <file>` | Folder: entry HTML relative to the folder (default `index.html`) |
| `--workspace <id>` | Check as an upload to this workspace |
| `--json` | Force JSON output |

`data` fields: `kind`, `mode_effective`, `detected_slides`, `size_ok`, `size_bytes`,
`size_limit_bytes`, `title`, `warnings`, `images_extracted`, `external_resource_hosts`,
`scripts_would_run`, `same_title_pages`, `would_change`, and `files` for a folder. When the
size is refused before the checks run, `data` is only `size_ok: false`, `size_bytes`,
`size_limit_bytes`, `would_change: true`, `warnings: []` and `message`.

Exit `0`: it would publish as it is. Exit `1` with `data`: `would_change` is true, meaning
the size is over the cap or a warning is `removed_tag`, `removed_attribute` or
`external_refs_block_scripts`. `images_extracted` alone does not set it.

## Upload

```bash
sharedrop upload <path> [options]          # alias: sharedrop drop
```

`<path>` is a file, `-` for stdin, or a folder. Files: HTML, MHTML, Markdown, PDF and
images (PNG, JPEG, WebP, GIF, AVIF, BMP, ICO, APNG, SVG, HEIC/HEIF, TIFF). Also JSON and
JSONL, CSV, DOCX, XLSX, plain text, source code files and video (MP4, WebM, MOV, M4V);
images and video need a paid plan (a free account gets `TIER_LIMIT`). A folder
uploads as a multi-file bundle: an entry HTML plus relative css, js, image and font files.
Hidden files and unsupported types (a `README.md`, for example) are left out and listed
in `skipped`.

| Flag | Meaning |
|---|---|
| `--title <title>` | Page title (detected from the file if omitted) |
| `--visibility <vis>` | `private` (default), `shared`, `public` |
| `--mode <mode>` | HTML only: `static` or `interactive`. Default: the account default for a new page (interactive unless the owner changed it); a re-upload keeps the page's mode |
| `--entry <file>` | Folder uploads: entry HTML relative to the folder (default `index.html`) |
| `--page-id <id>` | Replace an existing page's content and keep its URL |
| `--folder <id or path>` | File the new page in a folder (Pro), single file or bundle; missing segments are created |
| `--to <slug or id>` | Claim a reserved address made with `sharedrop reserve` (single files only) |
| `--workspace <id>` | Upload to a workspace |
| `--json` | Force JSON output |

## Upload and update JSON

`upload` and `update <id> <file|folder>` print the same `data` shape. This is the CLI's
output for a new page, copied from its test contract:

```json
{
  "data": {
    "id": "3f0c2b4e-8f7a-4c1d-9e2b-1a2b3c4d5e6f",
    "title": "Q3 report",
    "url": "/scottoau/ab12cd34ef",
    "full_url": "https://sharedrop.cloud/scottoau/ab12cd34ef",
    "kind": "html",
    "mode": "interactive",
    "visibility": "private",
    "was_reupload": false,
    "version": 1,
    "scripts_will_run": false,
    "external_resource_hosts": ["cdn.jsdelivr.net"],
    "warnings": [
      { "code": "images_extracted", "detail": "img", "count": 2, "message": "2 inline images were moved to hosted storage." },
      { "code": "external_refs_block_scripts", "detail": "cdn.jsdelivr.net", "count": 1, "message": "Scripts will not run because the page loads resources from 1 external host. Turn on external network for this page or vendor the files." }
    ],
    "same_title_pages": [
      { "id": "7d1e0a9c-2b3f-4e5d-8c7b-6a5f4e3d2c1b", "full_url": "https://sharedrop.cloud/scottoau/zz98yy76xx", "updated_at": "2026-10-06T22:14:03.000Z" }
    ]
  }
}
```

- `version` is 1 on a new page and goes up by one on each revision; a revision keeps the
  same `id` and `url` and has `was_reupload: true`.
- `scripts_will_run` reflects the page's current settings; an owner toggle can change it.
- `warnings` is always present. Codes: `removed_tag`, `removed_attribute`,
  `images_extracted`, `external_refs_block_scripts`. Branch on `code`, not `message`.
- `same_title_pages` appears on a new page only: other pages of yours with the same title.
  It is a hint for the reply, never a reason to update one of them.
- `skipped` appears on a folder bundle: `{ path, reason }` for each file left out.

## Update

```bash
sharedrop update <id> [file|folder] [options]
```

Pass a file or folder to replace the content (same URL, `version` up by one), and/or flags
to change metadata: `--title`, `--visibility`, `--mode` (when replacing content; default
keeps the page's mode) and `--slug <slug>` (Pro, public pages only; the old address
redirects). A folder must have `index.html` as its entry; for another entry use
`sharedrop upload <folder> --entry <file> --page-id <id>`.

Uploading again without the id creates a duplicate page and spends quota.

## Read back

```bash
sharedrop fetch <ref>                    # served root content to stdout
sharedrop fetch <ref> -o page.html       # or to a file
sharedrop download <ref> -o page.zip     # the whole artefact (root plus assets) as a zip
```

`fetch` works on your pages, public pages and pages shared with you, on every tier. It is
not byte-preserving for rendered kinds: Markdown and skill pages come back as sanitised or
rendered HTML, and inline `data:` images come back as hosted `/api/images/...` links.
Compare checksums only for kinds that keep their bytes; otherwise check `get` metadata and
look for the expected content. `download` of an archive page streams the raw file.

## Find and inspect

```bash
sharedrop list --json                    # your pages; --limit <n> (default 50), --cursor <id>
sharedrop list --folder reports/q3 --json
sharedrop search "q4 report" --json      # matches title, slug, id and file type
sharedrop get <ref> --json               # one page's metadata
```

List and search rows carry `id`, `slug`, `title`, `kind`, `mode`, `version`,
`visibility`, `full_url`, `created_at` and `updated_at`. `get` also carries
`scripts_will_run` and `external_resource_hosts`.

## Share and disappearing links

```bash
sharedrop share <ref> --email alice@example.com --json
```

On a paid tier the page becomes `shared`; on free it stays `private` and the recipient
opens it through the grant. Share emails have a daily limit; over it the share still
works but no email is sent, and `--json` output carries `email_warning`.

Disappearing links (Pro) are separate URLs with their own limits. They never change the
page's visibility or share list.

```bash
sharedrop link create <ref> --expires-in 12h --max-views 5 --json      # anyone holding it
sharedrop link create <ref> --people a@x.com,b@x.com --expires-in 7d --json
sharedrop link list <ref> --json                                       # views, limits, people
sharedrop link people <ref> <link-id> --add c@x.com --remove a@x.com --json
sharedrop link revoke <ref> <link-id> --json                           # page is unchanged
```

| `link create` flag | Meaning |
|---|---|
| `--people <emails>` | Only these people, after signing in (comma or space separated) |
| `--expires-in <duration>` | Time limit such as `30m`, `12h`, `7d` |
| `--max-views <n>` | Total views across everyone |
| `--present` | Slide decks: open straight into fullscreen Present mode |
| `--no-email` | With `--people`: do not email them; the user sends the link |

## Folders and trash

Folders need Pro. A free key gets `FOLDERS_RESTRICTED` (exit `3`) with an upgrade link.

```bash
sharedrop folder create reports/2026/q3 --json      # nested; creates missing segments
sharedrop folder list --json                        # top level; --parent <id> for children
sharedrop folder rename <id> "Q3 Reports" --json
sharedrop folder move <id> --root --json            # or --parent <id>
sharedrop folder restore <id> --json                # restore a folder or page from trash (30 days)
sharedrop move <id> --folder reports/q3 --json     # move a page in
sharedrop move <id> --root --json                  # or back to the top level
```

`sharedrop trash empty` permanently deletes everything in the trash. Run it only when the
user asks for exactly that.

## Large files

```bash
sharedrop archive big.zip --title "Raw data" --json    # Pro; zip, tar, gz, tgz, sql, sql.gz
```

Archives are private, download-only and count against storage. `--store-as-file` stores
any large file as a download-only blob. Use an archive when content is over the 10 MB HTML
and text cap and the user only needs to download it.

## Watermark

The CLI has no watermark flag. Turn the overlay on with the MCP tool
`Sharedrop:update_page` and `watermark_enabled` (see [mcp-and-rest.md](mcp-and-rest.md)),
or in the dashboard.

## Delete

```bash
sharedrop delete <exact-page-id> --json
sharedrop folder delete <exact-folder-id> --force --json   # contents go to trash
```

Only on an explicit request, with the target resolved to an exact id first (see SKILL.md,
"Delete only on request").
