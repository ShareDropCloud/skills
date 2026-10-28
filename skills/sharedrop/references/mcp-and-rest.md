# MCP and REST (no shell)

Use these only when you cannot run the `sharedrop` CLI. With a shell, the CLI does the
same work in one command. The rules in SKILL.md still apply: new page or revision
(step 2), visibility, delete only on request, report the exact URL.

## Contents

- [MCP tool names](#mcp-tool-names)
- [Upload with MCP](#upload-with-mcp)
- [Other MCP tools](#other-mcp-tools)
- [Connect the MCP server](#connect-the-mcp-server)
- [REST API](#rest-api)

## MCP tool names

Tool names below are written `Sharedrop:<tool>`. The prefix your host shows depends on
how the server was added:

| Host setup | Example name |
|---|---|
| claude.ai connector, used from Claude Code | `mcp__claude_ai_ShareDrop__create_upload` |
| Server added as `sharedrop` in an MCP config | `mcp__sharedrop__create_upload` |

Match on the tool suffix (`create_upload`, `finalize_upload` and so on) and use the full
name your host lists. Do not guess a prefix.

## Upload with MCP

Every file, HTML included, uses the same streamed flow. Base64 is never accepted. There is
no pre-check tool on MCP, so read `warnings` from the finalize result instead.

1. `Sharedrop:create_upload` returns `upload_url` and `upload_token`.
2. HTTP `PUT` the raw bytes to `upload_url` with `Authorization: Bearer <upload_token>`.
3. `Sharedrop:finalize_upload` publishes and returns `url`, `page_id`, `mode`,
   `visibility`, `kind`, `was_reupload`, `version`, `scripts_will_run`,
   `external_resource_hosts`, `warnings` and, on a new page, `same_title_pages`.
4. To revise a page and keep its URL, pass its `page_id` to both `Sharedrop:create_upload`
   and `Sharedrop:finalize_upload`; `version` goes up by one. Pass `folder_id` or
   `folder_path` to file a new page (Pro), and `slides: true` for a deck. Leave `mode`
   out to take the account default on a new page and keep the page's mode on a revision.
5. Multi-file bundles: call `Sharedrop:create_upload` and PUT once per file, then call
   `Sharedrop:finalize_bundle` once with every `{ path, object_key, upload_token }`.
   Exactly one path is `index.html`. To replace a bundle page, pass the same `page_id` to
   the `index.html` file's `Sharedrop:create_upload` and to `Sharedrop:finalize_bundle`.
   Sign the root with an HTML `filename` and each asset with a `filename` that has the
   same extension as its `path`, or `Sharedrop:finalize_bundle` refuses it with
   `file_name_mismatch`.
6. Team workspaces: when the user mentions a team or workspace, call
   `Sharedrop:list_workspaces` and pass the matching `slug` (the workspace address) as
   `workspace` to `Sharedrop:create_upload` and `Sharedrop:finalize_upload`. Never ask the
   user for a workspace id. A revision of a workspace page needs only `page_id`. Check the
   workspace's `can_upload` first (false for a read-only viewer), and do not edit someone
   else's page unless `can_edit_all_pages` is true (owners and members; both edit every page
   in the workspace). `can_upload` reflects your role; a workspace with a billing problem refuses uploads with 402 until the owner fixes billing.

Call `Sharedrop:whoami` when you need the username, tier or entitlements (images and
video need a paid plan on MCP uploads too). Verify from the finalize result as in SKILL.md
step 5; `Sharedrop:get_page` returns metadata including `version` and `scripts_will_run`,
and `Sharedrop:fetch_page` returns a short-lived `fetch_url` to GET with no auth header.

## Other MCP tools

| Job | Tool |
|---|---|
| List or inspect | `Sharedrop:list_pages`, `Sharedrop:get_page` |
| Team workspaces | `Sharedrop:list_workspaces` (each has `id`, `name`, `slug`, `role`, `tier`, `can_upload`, `can_edit_all_pages`); `workspace` on `list_pages`, `create_upload`, `finalize_upload`, `finalize_bundle`, `list_folders`, `create_folder`, `create_reservation` and `list_reservations` takes the id or address |
| Past versions (your own pages, every page in your workspaces) | `Sharedrop:list_versions` (current first; each has `version`, `selector`, `live`, `kind`, `file_size`, `has_original`), then `Sharedrop:fetch_page` with `version` (a number or `"latest"`) |
| Metadata, watermark | `Sharedrop:update_page` (`watermark_enabled` for the overlay) |
| Share by email | `Sharedrop:share_with_email`, `Sharedrop:share_page`, `Sharedrop:list_shares`, `Sharedrop:revoke_share` |
| Pages others shared with you | `Sharedrop:list_shared_with_me` (newest first; each has `page_id`, `title`, `kind`, `url` (path), `full_url` (link to give the user), `shared_by`, `shared_at`, `expires_at`; `limit` and `cursor` to page). Read one with `Sharedrop:fetch_page`. A workspace-scoped key is refused |
| Disappearing links (Pro) | `Sharedrop:create_ephemeral_link`, `Sharedrop:list_ephemeral_links`, `Sharedrop:update_ephemeral_link`, `Sharedrop:revoke_ephemeral_link` |
| Folders (Pro, or Team in a workspace) | `Sharedrop:create_folder`, `Sharedrop:list_folders`, `Sharedrop:move_page`, `Sharedrop:delete_folder`, `Sharedrop:restore_page`. `move_page` and `delete_folder` act in the page's or folder's own space; with `workspace` on `finalize_upload`, `folder_path` resolves in that workspace's folders |
| Delete (explicit request only) | `Sharedrop:delete_page` |

Use the version tools when the user wants an earlier version back, wants to compare two
versions, or a re-upload went wrong. `Sharedrop:fetch_page` with `version` returns a
`fetch_url` that serves that version's exact upload as a download (a folder upload as a
zip), plus `version`, `filename` and `content_type`. Only a version with
`has_original: true` has one. The URL lasts about 5 minutes and works for anyone holding
it, so do not paste it anywhere public. To recover, re-upload the good file with the
page's `page_id`: same link, new version number. A page only shared with you returns
`PAGE_NOT_FOUND`.

`Sharedrop:create_ephemeral_link` takes `page_id` plus `expires_in_seconds`, `max_views`
or both (at least one is required), `audience: "anyone"` or `audience: "people"` with
`emails`, `notify: false` to skip the email, and `present_only: true` for a deck that
opens fullscreen.

## Connect the MCP server

Remote HTTP only, at `https://sharedrop.cloud/api/mcp`, with OAuth on first connect or a
Bearer `sd_` key:

```json
{
  "mcpServers": {
    "sharedrop": { "type": "http", "url": "https://sharedrop.cloud/api/mcp" }
  }
}
```

Per-client setup: https://sharedrop.cloud/dashboard/settings/mcp

## REST API

The last resort, with neither a shell CLI nor MCP. Upload is a streamed three-step flow
for any file type: sign, PUT the bytes, finalize. Keep the key in `SHAREDROP_TOKEN`; never
paste it into a command.

```bash
# 1. sign: reserve a key and mint a 5-minute upload token
SIGN=$(curl -s -X POST https://sharedrop.cloud/api/upload/sign \
  -H "Authorization: Bearer $SHAREDROP_TOKEN" -H "Content-Type: application/json" \
  -d '{"filename":"report.html","content_type":"text/html","size_bytes":'"$(wc -c <report.html)"'}')
UPLOAD_URL=$(echo "$SIGN" | jq -r '.upload_url')
UPLOAD_TOKEN=$(echo "$SIGN" | jq -r '.upload_token')
OBJECT_KEY=$(echo "$SIGN" | jq -r '.object_key')

# 2. PUT the raw bytes straight to storage (no request-body size cap)
curl -X PUT "$UPLOAD_URL" \
  -H "Authorization: Bearer $UPLOAD_TOKEN" -H "Content-Type: text/html" \
  --data-binary @report.html

# 3. finalize: sanitise and publish; add "page_id":"<id>" to sign and finalize to revise
curl -X POST https://sharedrop.cloud/api/upload/finalize \
  -H "Authorization: Bearer $SHAREDROP_TOKEN" -H "Content-Type: application/json" \
  -d '{"object_key":"'"$OBJECT_KEY"'","upload_token":"'"$UPLOAD_TOKEN"'","title":"Q4 Report","visibility":"private"}'

# Team workspaces: the id and address (slug) of each; pass either as "workspace" to sign
# and as "workspace_id" to finalize and GET /api/v1/pages
curl -s -H "Authorization: Bearer $SHAREDROP_TOKEN" https://sharedrop.cloud/api/v1/workspaces

# Pages other people shared with you, newest first (personal key only; ?cursor=&limit= to page)
curl -s -H "Authorization: Bearer $SHAREDROP_TOKEN" https://sharedrop.cloud/api/v1/shared

# Read a page's raw content (two-step token handoff)
FETCH_URL=$(curl -s -H "Authorization: Bearer $SHAREDROP_TOKEN" \
  https://sharedrop.cloud/api/v1/pages/<page_id>/fetch | jq -r '.data.fetch_url')
curl -s "$FETCH_URL" -o page.html      # no auth header; the URL token is the credential

# Past versions (your own pages, every page in your workspaces): list them, then download one as uploaded
curl -s -H "Authorization: Bearer $SHAREDROP_TOKEN" \
  https://sharedrop.cloud/api/v1/pages/<page_id>/versions
curl -s -OJ -H "Authorization: Bearer $SHAREDROP_TOKEN" \
  https://sharedrop.cloud/api/v1/pages/<page_id>/versions/2/download
```

Full API reference: https://sharedrop.cloud/docs/api-reference
