# Changelog: sharedrop skill

## Contents

- [2026.10.07 (multi-file package for CLI 1.12.0)](#20261007-multi-file-package-for-cli-1120)
- [Changes from the previous public skill](#changes-from-the-previous-public-skill)
- [Changes from the audit rebuild](#changes-from-the-audit-rebuild)
- [Deviations kept on purpose](#deviations-kept-on-purpose)
- [Release record](#release-record)

## 2026.10.07 (multi-file package for CLI 1.12.0)

The skill is now a folder: `SKILL.md`, six files under `references/` linked directly from
SKILL.md, and this changelog. It merges the previous public skill (one file,
`sharedrop-skill.md`) with the audit rebuild of 6 October 2026, applies the review items
S1 to S10 from that audit, and teaches what `@sharedrop/cli` 1.12.0 shipped (GitLab
sharedrop #383 and #384). Every publishing surface ships the whole folder: the GitHub
mirror (`skills/sharedrop/`), `sharedrop-skill.zip`, `skill.sh` and the files served under
`/sharedrop-skill/`. `/sharedrop-skill.md` still serves `SKILL.md`.

## Changes from the previous public skill

### Structure

- One 399-line file became `SKILL.md` (308 body lines) plus `references/cli-reference.md`,
  `html-pages.md`, `slides.md`, `skill-pages.md`, `mcp-and-rest.md` and
  `model-support.md`, each linked from SKILL.md with when to read it. Files over 100 lines
  have contents lists.
- Description rewritten in the third person with what, when and near-miss exclusions;
  `compatibility` and `model: claude-sonnet-5-5` added to the frontmatter.
- The publish path is a six-step checklist with a degree of freedom on every step, a
  return path from a failed check and a limit of two repairs.
- A `## Recommended model` section and a support record.

### Behaviour

- **New page or revision (S1, owner decision 2).** A document uploaded earlier in the
  session, or a page the user names by id or URL, is a revision of that page: same id,
  same link, `version` up by one. Revision wording with no id searches by title. Anything
  else is a new page; a page is never updated because its title matches, and
  `same_title_pages` is mentioned in one line. The old text said "search for the title
  first" and "update the existing page by id when it is a revision", which let a title
  match become an overwrite.
- **Interactive is the default mode (#383, owner decision 3).** The old text said the CLI
  defaults to `static`. With no `--mode` a new page takes the account default
  (interactive unless the owner changed it) and a revision keeps the page's mode.
- **`sharedrop check` before upload (#383).** Exit 1 with `data` means something would be
  removed or blocked. Each warning code has an action.
- **New JSON fields (#383):** `version`, `was_reupload`, `scripts_will_run`,
  `external_resource_hosts`, `same_title_pages`, `skipped` and the `warnings` codes
  `removed_tag`, `removed_attribute`, `images_extracted`, `external_refs_block_scripts`.
- **Exit codes (#383, S8):** `2` no token found, `3` token rejected or action forbidden,
  and `7` plan or billing limit, which the old list left out.
- **`update <id> <folder>` and `--folder` on bundles (#383).**
- **Verify from the response (S4).** The upload response is the first proof; served
  content is fetched only when the request depends on it, in one command. `whoami` runs
  only when the job needs an account fact.
- **Inline images (S3).** Inline `data:` images are moved to hosted storage and still
  show. The old advice to inline small images stays true; the rebuild's rule that they are
  stripped is dropped.
- **Custom domain (S5):** `full_url` can be on the owner's own domain.
- **Secrets (S6):** report the variable name and line only; never any part of a value.
- **CDN libraries (S7):** a vendoring recipe and a stop rule.
- **`shared` visibility:** used only when `whoami` lists it in
  `entitlements.allowedVisibilities`.
- **REST sign example:** reads `upload_url`, `upload_token` and `object_key` from the top
  level of the sign response, which has no `data` wrapper. The old example read
  `.data.upload_url` and got `null`.
- Error table: `TIER_LIMIT` now covers the page cap (the server's code since #383);
  `PAGE_NOT_FOUND`, `STORAGE_LIMIT` and `FOLDERS_RESTRICTED` rows added.
- REST examples read the key from `$SHAREDROP_TOKEN` instead of a pasted `sd_YOUR_KEY`.

## Changes from the audit rebuild

Applied from the audit review (`08-fable-review.md`):

| Item | Change |
|---|---|
| S1 | Step 2 rewritten: revise only on an id or URL in hand (including a page uploaded this session) or revision wording; never on a title match |
| S2 | Dropped the "`<` inside `<style>` drops the block" rule and `fix-style`: the sanitiser keeps the block since 31 Aug 2026 |
| S3 | Dropped "base64 `data:image` URIs are stripped" and the forced bundle; images are extracted and verified by the served `<img>` count |
| S4 | Step 5 reads the upload response first (`warnings`, `version`); `whoami` only when needed; served check in one command |
| S5 | One sentence on `full_url` on the owner's domain (step 6) |
| S6 | Secrets: name and line only, never any part of the value (step 1) |
| S7 | `references/html-pages.md` "Vendor a CDN library" with a stop rule |
| S8 | `fetch` and `download` take a slug or URL; exit 3; `--folder` on bundles; `whoami` fields |
| S9 | `mode: interactive` does not prove scripts run; `scripts_would_run` and `scripts_will_run` do. The checker that disagreed with the product is gone |
| S10 | Watermark and archive moved to `references/cli-reference.md`; the instruction to copy the checklist into the reply is dropped (the checklist stays) |

Also removed:

- `scripts/html_check.py`. `sharedrop check` now runs the product's own rules (sanitiser,
  image extraction, slides detection, size cap, external hosts), so a local copy of those
  rules is no longer needed and had already gone out of date twice (S2, S3). Its last
  remaining job, counting served images, is one `grep`. The `python3` requirement goes
  with it.
- "Old patterns" keeps the retired `POST /api/v1/pages` and adds the pre-1.12 CLI
  behaviour (static default, no folder `update`, `--folder` ignored on bundles, exit 2 on
  a rejected token).

Kept from the rebuild: the 35 rule sentences mapped in its `checks/rules-map.tsv`, with
two deliberate changes: "Update the existing page by id when it is a revision" is now
step 2's rule set (S1), and "Mode ... defaults to static" is now the account default
(#383).

## Deviations kept on purpose

- **D-1. `model` key fails skill-creator's `quick_validate.py`.** Its allowed keys are
  name, description, license, allowed-tools, metadata and compatibility, so it reports an
  unexpected key. The Ascend combined skill audit (section K) directs `model:` in the
  frontmatter for a Claude Code skill, and Claude Code reads it. The audit wins; the same
  file without the line passes the validator.
- **D-2. `model:` is set although it did not switch the model in a test.** On Claude Code
  2.1.292 in headless `claude -p`, Opus served every turn after the skill loaded. The
  field stays because the audit requires the documented mechanism; SKILL.md says it did
  not switch the model and tells the user to choose Sonnet with `--model` or `/model`.
- **D-3. Search kept for revision wording without an id.** Review item S1 drops inference
  from search results; the search stays only when the user asked for a revision and named
  no page, so case "here is the new version" still finds its page.
- **D-4. Recommendation carried over as provisional.** The model results come from the
  previous version; this version's eval rerun is pending (GitLab sharedrop #384).
- **D-5. Not documented:** `reserve` and `reservations` beyond the `--to` flag, workspaces
  beyond the flag, and `about`. They were not in either source skill's scope.
- **D-6. Product name.** "Sharedrop" in the text; the description also says "ShareDrop"
  because the claude.ai connector writes it that way, which helps selection.

## Release record

| Item | Where |
|---|---|
| Previous versions | Previous public skill: `public/sharedrop-skill.md` in the Sharedrop app repository before #384, and ShareDropCloud/skills history. Audit rebuild: Ascend skills repository, `sharedrop/` |
| Setup notes | SKILL.md "Before you start" and `compatibility`; `references/cli-reference.md` (install, sign-in order) |
| Support matrix and model recommendation | `references/model-support.md`; short form in SKILL.md `## Recommended model` |
| Evidence | Audit `sharedrop-2026-10-06` (`04-evaluation-report.md`, `05-decision.md`, `08-fable-review.md`); eval rerun for this version pending |
| Hard rules and enforcement | Three rules with a material consequence rest on prose only: never update a page because its title matches, delete only on an explicit request, never print any part of a secret. No Claude Code hook is configured. The product adds hints, not blocks: `same_title_pages` on a new upload, `check` exit 1 before a change, and `version` and `was_reupload` to prove a revision. Other hosts have no enforcement beyond the prose and their own permission prompts |
| Publishing | `scripts/sync-skills-repo.sh` mirrors this folder to `skills/sharedrop/` in ShareDropCloud/skills on every production promote, after the eval rerun passes |
