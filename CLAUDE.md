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

## Diagrams: SVG over ASCII art

When a blog post needs a diagram, flow chart, or visual explanation, create an **SVG file** in the post directory instead of using ASCII art in markdown code blocks.

- Name SVGs descriptively: `cmux-flow.svg`, `three-layer-fix.svg`, `input-buffer.svg`
- Use dark theme styling to match the blog aesthetic:
  - Background: `#1e293b` → `#0f172a` gradient
  - Text: `#e2e8f0` (primary), `#94a3b8` (secondary), `#64748b` (muted)
  - Accent borders: blue `#3b82f6`, amber `#f59e0b`, green `#22c55e`, red `#ef4444`
  - Monospace font stack: `'SF Mono', 'Fira Code', 'Cascadia Code', monospace`
- Reference in markdown as: `![alt text](filename.svg)`
- Small inline annotated snippets (e.g. a 4-line code breakdown) are fine as code blocks — SVGs are for structural/architectural diagrams

## Images

- Compress hero images to JPEG at 80% quality using `sips` (built-in macOS tool)
- Target size: under 300KB for hero images
- Command: `sips -s format jpeg -s formatOptions 80 input.png --out hero.jpg`
