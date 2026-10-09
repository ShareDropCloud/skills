# HTML pages: modes, bundles, images and served content

Read this when `sharedrop check` reports a finding, or the page has scripts, images or
files it loads from elsewhere.

## Contents

- [Static or interactive](#static-or-interactive)
- [Interactive pages must be self-contained](#interactive-pages-must-be-self-contained)
- [Vendor a CDN library](#vendor-a-cdn-library)
- [Images](#images)
- [Folder bundles](#folder-bundles)
- [Check the served page](#check-the-served-page)

## Static or interactive

With no `--mode`, a new page takes the account's default mode, which is interactive unless
the owner changed it, and a revision keeps the page's current mode. `sharedrop check`
prints the mode it would use as `mode_effective`.

- **Interactive** runs the page's own JavaScript (tabs, filters, charts, count-ups) in a
  sandbox, as long as the page loads nothing external.
- **Static** strips every script; `check` and the upload report each removal as a
  `removed_tag` warning. It suits a report with no scripts, and CSS transitions still work.
  Pass `--mode static` when a page carries scripts that must not run.

`mode: interactive` in a response does not prove the scripts run. `scripts_would_run`
(from `check`) and `scripts_will_run` (from `upload`, `update` and `get`) do, for the
page's current settings; the owner can still change those settings later.

## Interactive pages must be self-contained

Interactive pages run in a locked-down, offline sandbox on a separate origin with a strict
content security policy. If the page loads *anything* external (a CDN script or
stylesheet, a Google Font, a remote image, video or iframe, a CSS `url()` on another host,
or a `<base href>`), Sharedrop disables **all** of its JavaScript and serves it static
with a banner. Links (`<a href>`) and URLs quoted in text or script do not count. `check` and the upload
report this as `external_refs_block_scripts`, list the hosts in `external_resource_hosts`
and set `scripts_would_run` / `scripts_will_run` to `false`.

Build interactive pages closed:

- CSS inline in `<style>`, JavaScript inline in `<script>`.
- No `fetch()` of remote data at runtime. Drive tabs, filters and charts from data inlined
  in the page.
- Libraries, fonts and media either inline or as relative files in a folder bundle.

A page that truly needs the open internet works only if the human owner enables external
network access for it in the dashboard; an agent cannot grant that to itself.

## Vendor a CDN library

When `check` reports `external_refs_block_scripts` for a library such as Chart.js, ship a
copy of the library with the page instead of rewriting the page:

```bash
mkdir -p bundle/vendor
curl -fsSL https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js -o bundle/vendor/chart.umd.min.js
cp page.html bundle/index.html
# In bundle/index.html: point <script src> at vendor/chart.umd.min.js, and remove or
# vendor every other host check listed (for example a Google Fonts <link>).
sharedrop check bundle/ --json     # expect exit 0 and scripts_would_run: true
sharedrop upload bundle/ --title "Report" --json
```

Use the exact URL and version the page already loads. If you cannot download the file (no
network, or the host blocks the download), stop and ask the user. Do not replace the
library with hand-written code or swap the chart for something else: that changes the
deliverable.

## Images

Inline `<img src="data:image/...;base64,...">` images are fine in a single HTML file:
Sharedrop moves them to hosted storage and rewrites each `src`, and the upload reports
`images_extracted` with the count. They still show; nothing is lost.

Images the extractor does not move (CSS `background-image: url(data:...)`, `srcset`,
`<picture><source>`, `<svg><image>`) stay inline and count toward the 10 MB HTML cap. When
a page is near the cap, or the images are large, put them in a folder bundle as files.

## Folder bundles

A folder uploads as one page: an entry HTML (default `index.html`) plus relative css, js,
image and font files. Use one when the page has large images, vendored libraries or fonts,
or more than one file.

```bash
sharedrop check bundle/ --json
sharedrop upload bundle/ --title "Report" --json            # --entry <file> for another entry
sharedrop update <id> bundle/ --json                        # revise: same URL, version up
```

Hidden files and unsupported types are left out and listed in `skipped`; check that no file
the page needs is there. `--folder` works on bundles. `update <id> <folder>` always uses
`index.html` as the entry; for another entry, revise with
`sharedrop upload bundle/ --entry <file> --page-id <id> --json`.

## Check the served page

The response's `warnings` already say what the server removed or moved. Fetch the served
bytes only when the request depends on something the response does not show, and keep it
to one command:

```bash
# Single file: images served versus images in your file.
sharedrop fetch <id> -o served.html && grep -o '<img' served.html | wc -l && grep -o '<img' page.html | wc -l

# Folder bundle: list the files Sharedrop holds.
sharedrop download <id> -o page.zip && unzip -l page.zip
```

If a count is short, read the `warnings` and `skipped` from the upload, fix the file and
revise the same page by id.
