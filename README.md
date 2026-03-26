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

Push to the `publish` branch and [**cr-engine**](https://github.com/nirus/cr-engine) picks up the content, builds the pages, and serves them at [coder.rocks](https://coder.rocks). No framework config, no build step in this repo — just content.

## Writing a New Post

1. Create a directory with a kebab-case slug
2. Add `claim.json`, `index.md`, and a `hero.jpg`
3. Push to `publish` — [cr-engine](https://github.com/nirus/cr-engine) handles the rest

See [`CLAUDE.md`](CLAUDE.md) for content conventions (SVG diagrams, image compression, mobile compatibility).

## License

Content is published at [coder.rocks](https://coder.rocks).
