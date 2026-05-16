# vault-graph

Force-directed canvas viewer for an Obsidian-style markdown vault, with
a side panel that renders any clicked note. Drop it on top of a
precomputed `graph.json` + a folder of markdown and you have an
interactive map of your notes; provide write callbacks and you have a
collaborative editor.

```
                  ┌──────────────────────────┐
                  │  vault-graph (this repo) │
                  │  dist/vault-graph.js     │
                  │  dist/vault-graph.css    │
                  └────────┬─────────────────┘
                           │ vendored copy
            ┌──────────────┴──────────────┐
            ▼                             ▼
   static consumer                  collaborative consumer
     (static fetch,                  (API fetch,
      read-only)                      edit + comments + auth)
```

## Use

```html
<link rel="stylesheet" href="vault-graph.css">
<script src="https://cdn.jsdelivr.net/npm/d3@7/dist/d3.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/marked@12/marked.min.js"></script>
<script src="vault-graph.js"></script>
<script>
  VaultGraph.create({
    brand: { title: 'my vault', blurbHTML: 'Click a node to read.' },
    fetchGraph: () => fetch('graph.json').then(r => r.json()),
    fetchNote:  (path) => fetch('vault/' + path).then(r => r.text()),
  });
</script>
```

## API

`VaultGraph.create(opts)` returns `{ data, openNote(idOrNode), getActivePanel(), destroy() }`.

| Option              | Type                                         | Required | Notes                                              |
|---------------------|----------------------------------------------|----------|----------------------------------------------------|
| `rootEl`            | `HTMLElement`                                | no       | Where to mount; defaults to `document.body`.       |
| `brand`             | `{title, blurbHTML, footerHTML}`             | no       | HUD title, blurb, bottom-right footer.             |
| `fetchGraph()`      | `() => Promise<{nodes, links}>`              | **yes**  | `nodes: [{id, label, group, path, deg}]`.          |
| `fetchNote(path)`   | `(string) => Promise<string>`                | **yes**  | Returns raw markdown.                              |
| `hudExtras(hud)`    | `(HTMLElement) => void`                      | no       | Plugin point — append your own UI to the HUD.      |
| `onPanelOpen(p)`    | `(panel) => void`                            | no       | Decorate the reader panel each time a note opens.  |
| `onPanelClose()`    | `() => void`                                 | no       | Cleanup after the panel closes.                    |
| `layoutCacheKey`    | `string`                                     | no       | Bump to invalidate the localStorage layout cache.  |

`panel` exposes:

```ts
{
  node, raw, bodyEl, actionsEl, extrasEl,
  addAction(label, onClick, { primary?, title? }): HTMLButtonElement,
  setBodyHTML(html), setBodyMarkdown(md), restoreBody(),
  setExtras(htmlOrEl), close(), panTo(k?), renderMarkdown(md),
}
```

Buttons added via `addAction` are tagged with `data-vg-plugin="1"` and are
removed automatically when the panel closes or a different note opens —
no cleanup needed on the plugin side.

## Vault data shape

```json
{
  "nodes": [
    { "id": "overview",  "label": "Overview", "group": "architecture", "path": "architecture/overview.md", "deg": 4 },
    { "id": "rankings",  "label": "Rankings", "group": "domain",       "path": "domain/rankings.md",       "deg": 2 }
  ],
  "links": [
    { "source": "overview", "target": "rankings" }
  ]
}
```

The graph can be produced with `examples/static-vault/build_graph.py` (parses
both `[[wikilinks]]` and `[md](links.md)`).

## Wikilinks inside notes

The engine rewrites `[[note-id]]` and `[[note-id|alias]]` into clickable
anchors. Clicking them opens the target inside the same panel. Markdown
links (`[…](…)`) inside the rendered body are left as-is by default.

## Consumers vendor a copy

Each consumer keeps a vendored copy under `static/vendor/vault-graph/`
(refreshed via `scripts/sync-vault-graph.sh`). No npm, no submodule —
deploy stays independent per consumer.
