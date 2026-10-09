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

Call `Sharedrop:whoami` when you need the username, tier or entitlements (images and
video need a paid plan on MCP uploads too). Verify from the finalize result as in SKILL.md
step 5; `Sharedrop:get_page` returns metadata including `version` and `scripts_will_run`,
and `Sharedrop:fetch_page` returns a short-lived `fetch_url` to GET with no auth header.

## Other MCP tools

| Job | Tool |
|---|---|
| List or inspect | `Sharedrop:list_pages`, `Sharedrop:get_page` |
| Metadata, watermark | `Sharedrop:update_page` (`watermark_enabled` for the overlay) |
| Share by email | `Sharedrop:share_with_email`, `Sharedrop:share_page`, `Sharedrop:list_shares`, `Sharedrop:revoke_share` |
| Disappearing links (Pro) | `Sharedrop:create_ephemeral_link`, `Sharedrop:list_ephemeral_links`, `Sharedrop:update_ephemeral_link`, `Sharedrop:revoke_ephemeral_link` |
| Folders (Pro) | `Sharedrop:create_folder`, `Sharedrop:list_folders`, `Sharedrop:move_page`, `Sharedrop:delete_folder`, `Sharedrop:restore_page` |
| Delete (explicit request only) | `Sharedrop:delete_page` |

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

# Read a page's raw content (two-step token handoff)
FETCH_URL=$(curl -s -H "Authorization: Bearer $SHAREDROP_TOKEN" \
  https://sharedrop.cloud/api/v1/pages/<page_id>/fetch | jq -r '.data.fetch_url')
curl -s "$FETCH_URL" -o page.html      # no auth header; the URL token is the credential
```

Full API reference: https://sharedrop.cloud/docs/api-reference
