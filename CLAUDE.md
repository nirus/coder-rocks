# coder-rocks blog

## Content structure

Each blog post is a directory at the repo root with a kebab-case slug:

```
<slug>/
  claim.json   — metadata (title, pubDate, description, tags, author)
  index.md     — article content (no frontmatter; metadata lives in claim.json)
  hero.jpg     — hero/cover image (jpg preferred for size; png acceptable)
```

Powered by `cr-engine`. Main branch is `publish`.

## Diagrams: Mermaid → PNG

When a blog post needs a diagram, flow chart, or visual explanation, use **Mermaid** (`.mmd` files) rendered to PNG. This avoids SVG XML pitfalls and ensures universal rendering.

**Workflow:**
1. Create a `.mmd` file in the post directory (e.g. `dream-lifecycle.mmd`)
2. Run `npm run diagrams` to render all `.mmd` files to `.png`
3. Reference in markdown as: `![alt text](filename.png)`

**Mermaid config** lives in `.mermaid/config.json` (dark theme, monospace font, blog color palette). The `diagrams` script uses it automatically.

**Styling notes (match coder.rocks theme):**
- Background: `#0a0a0a` (site body)
- Node fill: `#22272e` (code block bg) or `#262626` (inline code bg)
- Node border: `#404040`
- Text: `#e5e5e5` (primary), `#adbac7` (code text), `#666666` (muted)
- Use `style` directives in `.mmd` files for node-level coloring
- Output is 2x resolution for retina displays

**Commit both** the `.mmd` source and the `.png` output. The `.mmd` is the editable source of truth; the `.png` is what the blog displays.

Small inline annotated snippets (e.g. a 4-line code breakdown) are fine as code blocks — Mermaid diagrams are for structural/architectural visuals.

## Mobile compatibility

Site must render correctly on both desktop and mobile. The blog engine (cr-engine) does not add `overflow-x: hidden` or `max-width: 100%` on all elements, so content itself must stay within narrow viewports (~430px).

Common overflow culprits and fixes:
- **Code blocks**: Keep lines under ~50 characters. Use `\` line continuation for long statements. Use short inline comments (`# ^C` not `# Ctrl+C`). Remove trailing whitespace padding.
- **Markdown tables**: Minimize columns. Use short labels (`^A` not `Ctrl+A`). Drop columns that can be inferred. Move dense reference data to an appendix or a separate table.
- **Diagrams**: Use Mermaid → PNG pipeline (see Diagrams section). PNGs scale naturally and never overflow.

## Images

- Compress hero images to JPEG at 80% quality using `sips` (built-in macOS tool)
- Target size: under 300KB for hero images
- Command: `sips -s format jpeg -s formatOptions 80 input.png --out hero.jpg`

## Commits and PRs

Follow [Conventional Commits](https://www.conventionalcommits.org/). Allowed prefixes:

- `feat:` — new feature or blog post (triggers publish)
- `fix:` — bug fix (triggers publish)
- `chore:` — maintenance, config, dependency updates

PR titles must use the same format (e.g. `feat: add post on terminal multiplexers`). Merging a `feat:` or `fix:` PR to `publish` triggers a deploy.
