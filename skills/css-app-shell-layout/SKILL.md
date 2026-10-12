---
name: css_app_shell_layout
description: 'CSS pattern for a 100vh app shell where header and footer stay fully visible and only the middle div scrolls (flexbox or grid), including the min-height:0 gotcha and why sticky footer overlaps content'
tags:
- css
- flexbox
- grid
- layout
- sticky-footer
- scroll
created_at: '2026-09-23T10:35:34.387092885+02:00'
---

# CSS App-Shell Layout (100vh, header / scrolling main / footer)

Goal: outer div = 100vh, header and footer always fully visible, only `main` scrolls, footer stays at the bottom without overlapping content.

## Flexbox solution

```css
html, body { margin: 0; }

.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

header, footer {
  flex-shrink: 0; /* never squeezed by long content */
}

main {
  flex: 1;
  min-height: 0;    /* crucial: lets main shrink so it scrolls */
  overflow-y: auto; /* main is its own scroll container */
}
```

## Grid alternative

```css
.app {
  height: 100vh;
  display: grid;
  grid-template-rows: auto minmax(0, 1fr) auto;
}
main { overflow-y: auto; }
```

## Key insights

- Do NOT use `position: sticky` for the footer: sticky only works inside a scrolling container and makes content scroll *under* it (overlap). Instead, put the footer OUTSIDE the scroll area — only `main` scrolls.
- `min-height: 0` (flex) or `minmax(0, 1fr)` (grid) is the critical fix: flex/grid items refuse to shrink below their content size by default, so without it the footer gets pushed off-screen instead of `main` scrolling.
- `flex-shrink: 0` on header/footer prevents them from being compressed when content is long.
