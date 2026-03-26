<p align="center">
  <img src="banner.jpg" alt="CoderRocks — The Agentic Engineering Blog" width="100%" />
</p>

<h1 align="center">CoderRocks</h1>

<p align="center">
  <strong>The Agentic Engineering Blog</strong><br/>
  Deep dives into terminals, AI tooling, and the code that makes them tick.
</p>

<p align="center">
  <a href="https://coder.rocks">coder.rocks</a>
</p>

---

## How It Works

This repo is a **content-only** repository. Each blog post is a self-contained directory at the root:

```
<post-slug>/
  claim.json   — title, date, description, tags, author
  index.md     — article body (standard Markdown)
  hero.jpg     — cover image
```

Push to the `publish` branch and [**cr-engine**](https://coder.rocks) picks up the content, builds the pages, and serves them at [coder.rocks](https://coder.rocks). No framework config, no build step in this repo — just content.

## Posts

| Post | Tags |
|------|------|
| [Why Ctrl+C Prints Junk in CMUX — and How to Fix It](cmux-ctrl-c-fix-kitty-keyboard-protocol/) | terminal, zsh, CMUX, Ghostty |
| [Claude Opus 4.6 — What Changed and Why It Matters](claude-opus-4-6-transition-from-opus-4-5/) | AI, Claude, Agentic Coding |
| [Apollo GraphQL Client — Abort Pending Requests](apollo-graphql-client-abort-pending-requests/) | JavaScript, GraphQL |
| [Reverse Engineering Popunder.js in Chrome](reverse-engineering-popunder-js-chrome/) | JavaScript, Chrome, Security |
| [Storybook.js Custom Webpack Setup for SCSS](storybook-js-custom-webpack-setup-for-scss/) | JavaScript, Storybook, Webpack |

## Writing a New Post

1. Create a directory with a kebab-case slug
2. Add `claim.json`, `index.md`, and a `hero.jpg`
3. Push to `publish` — cr-engine handles the rest

See [`CLAUDE.md`](CLAUDE.md) for content conventions (SVG diagrams, image compression, mobile compatibility).

## License

Content is published at [coder.rocks](https://coder.rocks).
