# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal Hugo blog using the [bearblog](https://github.com/janraasch/hugo-bearblog) theme as a git submodule at `themes/hugo-bearblog/`. Hugo version is pinned via mise (`hugo-extended = "0.161.1"`). Deploys to Cloudflare Workers (Static Assets) via the `wrangler.jsonc` config in repo root; Cloudflare Workers Builds runs `hugo --gc --minify` then `npx wrangler deploy` on every push to `master`.

The `TODO` file at repo root tracks pre-launch tasks.

## Commands

```bash
hugo server                  # dev server with live reload; URL: http://localhost:1313/
hugo --gc --minify           # production build → public/
hugo new blog/<slug>.md      # scaffold a post from archetypes/default.md (creates draft)
```

The `hugo server` watcher writes to `public/` with `baseURL=http://localhost:1313/`, which silently overwrites any prior `hugo --gc --minify` output. To inspect a true production build (real `baseURL`, correct sitemap/RSS URLs), stop the server first then `rm -rf public/ && hugo --gc --minify`.

## Customizations

The theme submodule is never edited in place — all customizations are project-layer files that win Hugo's lookup chain over `themes/hugo-bearblog/`.

### `hugo.toml` divergences from bearblog's example config

| Key | What | Why |
|---|---|---|
| `title`, `author`, `copyright`, `description` | Site identity strings | User-specific. |
| `languageCode = "ru-RU"` | Changed from theme example's `"en-US"` | Russian-language blog. |
| `disableKinds = ["taxonomy"]` + `ignoreErrors` | Kept from theme example | Disables only the `/tags/` index page (Hugo 0.73+ semantics); individual term pages still render. We have no UI linking to `/tags/`. |
| `[permalinks] blog = "/:contentbasename/"` | Changed from theme's `/:slug/` | Flat root URLs derived from the post's filename (without extension). Default `:slug` falls back to a slugified `title`, which for Russian titles produces Cyrillic URLs (encoded as `%D0%9F...` when copied) — `:contentbasename` keeps URLs ASCII without needing `slug:` in every post's frontmatter. Note: `:filename` is the older spelling for the same token, deprecated in Hugo 0.144 and removed since. |
| `[permalinks] tags = "/tags/:slug/"` | Changed from theme's `/blog/:slug` | Tag URLs in their own namespace; no visual collision with post URLs. |
| `[[menu.main]]` About → `https://rgeraskin.dev` | Added | External link in header nav at weight 10 (after Home weight 1). |
| `hideMadeWithLine = true` | Enabled | Suppresses the "Made with Hugo Bear" footer link from the theme. |
| `enablePostNavigator = true` | Enabled (was commented out in example) | Renders the bottom navbar on every post. |
| `dateFormat = "2006-01-02"` | Enabled (was commented out in example) | ISO date format overrides theme default `"02 Jan, 2006"`. |

### Layout overrides

Each file at `layouts/<path>` shadows the corresponding `themes/hugo-bearblog/layouts/<path>`.

| Override | What | Why |
|---|---|---|
| `layouts/index.html` | Renders `_index.md` content, then tag cloud, then post list (`where .Site.RegularPages "Section" "blog"`) | Theme version is `{{ define "main" }}{{ .Content }}{{ end }}` — no posts on homepage. We want the homepage to be the post archive. |
| `layouts/_default/single.html` | Renders post title → date → **tags** → content → optional **discuss links** → post navigator | Theme renders tags after the post navigator at the bottom of the page; we moved them up next to the date. `{{ with .GetTerms "tags" }}` guard avoids empty `<p>` on non-blog single pages. Discuss block reads `.Params.discuss_tg` / `.Params.discuss_x` from frontmatter and renders only if at least one is set. |
| `layouts/partials/footer.html` | Emits `{{ .Site.Copyright }}` plus the conditional made-with line | Theme's footer renders only the made-with line. Bearblog never surfaces `.Site.Copyright` anywhere visible — we add it. |
| `layouts/partials/post_navigator.html` | Renders `<< Previous Post \| Home \| Next Post >>` | Theme version is `<< Previous Post \| Next Post >>` only. Home injected in the middle for one-click escape back to the post archive. |
| `layouts/partials/icon-telegram.html`, `layouts/partials/icon-x.html` | Inline SVG brand icons (24×24 viewBox, `currentColor` fill, `1em` size) for Telegram and X | Used by `single.html`'s discuss block. Inline SVG, no external requests, inherits link color. Brand marks from [simple-icons](https://simpleicons.org) (MIT). |
| `layouts/partials/custom_head.html` | Injects custom `<style>` block: blockquote outdent override (no outer indent, 2px left bar aligned with text edge); `content img` centered via `display: block` + `margin: auto` | Theme provides this partial empty for user CSS overrides. Using it instead of forking `partials/style.html` keeps the theme submodule update-safe. `baseof.html` loads `custom_head.html` after `style.html`, so on equal-specificity selectors our rules win the cascade. |

### Content & structure customizations

| Path | What | Why |
|---|---|---|
| `content/blog/_index.md` | `[build] render = "never"` | Suppresses the `/blog/` section index page so it doesn't duplicate the homepage. Child posts and term pages still render. `/blog/index.xml` is also gone; subscribers use `/index.xml` (the home feed). |
| `content/_index.md` | `menu = "main"`, `weight = 1` | Adds "Home" to the header nav via frontmatter (paired with the config-defined "About" entry). |
| `archetypes/default.md` | Pre-fills `description: 'PLACEHOLDER'`, `tags: ["PLACEHOLDER"]`, and empty `discuss_tg: ''` / `discuss_x: ''` in YAML frontmatter (`---` delimiters) | Theme's example archetype is TOML and omits all four. Every new post ships YAML-flavoured with these fields ready to fill in. Empty string for the discuss URLs is treated the same as missing — no link renders until you set them. |

### Static assets

`static/` files are copied verbatim to the site root at build time:

- `static/favicon.ico` — served as `/favicon.ico` (browsers auto-fetch this)
- `static/images/favicon.png` (32×32) — referenced by `[params] favicon = "images/favicon.png"`; emitted as `<link rel="shortcut icon">`
- `static/images/share.png` (900×600) — referenced by `images = ["images/share.png"]` for OpenGraph/Twitter previews. 900×600 is non-standard; platforms expect 1200×630.

## Hugo gotchas this setup hits

### `disableKinds = ["taxonomy"]` only disables the index page

In Hugo 0.73+ this disables only the `/tags/` (all-tags index) page. Individual term pages at `/tags/<name>/` still render via `_default/list.html`. If you add a `/tags/` link anywhere, remove `disableKinds = ["taxonomy"]` first or it will 404. `.Site.Taxonomies.tags` is always queryable regardless — that flag suppresses rendering, not the in-memory page graph.

### Homepage `.Pages` returns sections, not posts

On the homepage (`Kind: "home"`), `.Pages` returns top-level sections (e.g., the `blog` section as a single node), **not** individual posts. To enumerate posts, use `where .Site.RegularPages "Section" "blog"`. Already handled in `layouts/index.html`.
