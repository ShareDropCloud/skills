# Sharing an agent skill as a page

A standard agent skill can be a folder with `SKILL.md` plus `references/`, `scripts/`,
`assets/` or agent metadata. A Sharedrop native skill page is one uploaded file.
Sharedrop recognises `SKILL.md`, `*.skill.md` and `*.skill` as skill pages.

## Decide what to upload

Inspect the skill folder first:

- `SKILL.md` is self-contained (no links to other files it needs): upload it directly.
- The skill depends on other files: do not upload `SKILL.md` alone. Make a self-contained
  distribution copy such as `<name>.skill` that embeds the needed reference material and
  replaces local relative links. Leave the original folder unchanged as the source.
- Do not present a viewer URL as an installable multi-file skill package. If the recipient
  needs the original folder, use a method that keeps the full tree.

## New page or revision

Apply step 2 of SKILL.md: a skill you uploaded earlier in this session, or one the user
names by id or URL, is a revision; with revision wording and no id, search for the title.
Otherwise it is a new page, even when another page has the same title.

```bash
# New self-contained skill (folders need Pro)
sharedrop upload ascend-notion-crm.skill --title "Ascend Notion CRM" \
  --visibility private --folder Skills --json

# Revision: same URL, version goes up
sharedrop update <id> ascend-notion-crm.skill --json
```

## Verify

The response must show `kind: "skill"`, the intended `visibility`, the exact `full_url`,
and for a revision the same `id` with `version` one higher. A successful `--folder` upload
is in that folder. To confirm the embedded reference content made it:

```bash
sharedrop fetch <id> -o rendered-skill.html
```

The fetched page is rendered HTML, so look for the expected workflow and reference text;
do not expect its checksum to match the Markdown source.
