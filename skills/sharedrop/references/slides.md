# Slide decks

Sharedrop presents HTML decks fullscreen, so "make me a presentation" ends at a URL
rather than a PowerPoint export. Decks work on every plan, including Free. Full guide:
https://sharedrop.cloud/docs/slides

## Mark the page as a deck

There is no upload flag; the marker in the HTML makes any upload path produce a deck:

```html
<meta name="sharedrop:kind" content="slides" />
```

Use **one top-level `<section>` per slide**. Present shows one at a time and steps on
arrow key, space, click or tap:

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

## Presenter hooks (plain CSS you write)

- `data-sd-fragment` on an element inside a slide reveals it one step at a time. The next
  step moves to the next slide only once every fragment is shown. Fragments fade in by
  default; style the reveal off `__sd-frag-visible`.
- `__sd-active` is the class Present adds to the slide on screen. Key an entrance
  animation to it.

Present never restyles the deck; your CSS is what the audience sees.

## Check, upload, verify, present

```bash
sharedrop check deck.html --json      # expect data.detected_slides: true
sharedrop upload deck.html --title "Q3 review" --json
```

The upload response must show `kind: "slides"`. Present by adding `?present=1` to the
exact `full_url`. For an unattended screen, add `autoplay=<whole seconds>` to advance and
loop: `...?present=1&autoplay=15`.

A deck takes the same mode default as any HTML page (the account default, interactive
unless the owner changed it). Fragments, CSS transitions and slide-entry classes work in
either mode, because Present drives the slides. When the deck's own JavaScript must run
(count-ups, charts, canvas), confirm `scripts_will_run: true` and keep it self-contained
as described in [html-pages.md](html-pages.md#interactive-pages-must-be-self-contained).

For a link that opens straight into fullscreen on a TV (Pro):
`sharedrop link create <id> --present --expires-in 7d --json`.

On MCP (tool names in [mcp-and-rest.md](mcp-and-rest.md#mcp-tool-names)), pass
`slides: true` to `Sharedrop:finalize_upload` (ignored on non-HTML files), or rely on the
marker.
