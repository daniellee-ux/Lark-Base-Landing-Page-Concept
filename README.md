# Lark Base — Web Concepts

Working space for Lark Base website design explorations. Each subfolder is one self-contained concept; open its `index.html` directly in a browser — no build step.

## Concepts

| Folder | What it explores |
|---|---|
| [`landing-page/`](./landing-page) | Scroll-driven 4-section landing page. Premium section-coverage transitions built on GSAP ScrollTrigger + Lenis, with a live Tweakpane config panel for tuning the 16 feel parameters. See its [README](./landing-page/README.md) for the full design + technical breakdown. |

## How this repo is organized

- Each concept lives in its own folder with a top-level `index.html`.
- Concept folders are self-contained — their own assets, their own docs.
- Concepts are snapshots of exploration, not a shipping product. Expect things to be inconsistent between folders (different stacks, different conventions) — that's fine.

## Adding a new concept

1. Create a new folder: `<concept-name>/`
2. Drop in `index.html` (+ any assets)
3. Add a row to the table above
4. Commit — each concept is one commit or small series, with a message describing what it's exploring

## Live preview

If GitHub Pages is enabled on this repo, the structure maps to:

- `https://daniellee-ux.github.io/Lark-Base-Landing-Page-Concept/landing-page/` → landing-page concept
- `https://daniellee-ux.github.io/Lark-Base-Landing-Page-Concept/<folder>/` → other concepts as they land
