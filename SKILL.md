---
name: rheo-author
description: Author, configure, and build rheo projects. Use when working with rheo.toml, *.typ content files in a rheo project, or running rheo build/watch commands.
---

# rheo-author

Rheo is document infrastructure built on Typst. A single source tree compiles
**simultaneously** to PDF, HTML, and EPUB. This skill covers the `rheo.toml`
schema, project layout, and CLI. For raw Typst markup inside `.typ` files,
delegate to the `typst-author` skill.

## CLI

- `rheo init <dir>` — scaffold a new project (works in empty dirs; ignores `.git`/`.jj`).
- `rheo compile <path>` — one-shot build of all configured formats.
- `rheo watch <path> [--open]` — dev server with rebuild on change.
- Flags: `--config <path>`, `--build-dir <path>`.

`rheo init` scaffolds:

```
my-project/
├── rheo.toml
├── style.css
├── index.js
└── content/
    ├── index.typ        # uses target() show-rule for format-specific styling
    ├── about.typ
    ├── references.bib
    └── img/header.svg
```

## rheo.toml

Defaults (applied when keys are omitted):

```toml
version = "0.3.0"
content_dir = "./"
build_dir   = "build"
formats     = ["pdf", "html", "epub"]

[epub.spine]
title = "<project dir name>"
```

Restrict outputs by trimming `formats`, e.g. `formats = ["html", "epub"]`.

### Path resolution (load-bearing gotcha)

Once `content_dir` is set, `build_dir` **and all spine globs** resolve relative
to `content_dir`, not the project root. Asset `copy` paths stay relative to the
project root. Mix these up and files land in the wrong place.

## Spines — directory-scan default, exclude, sections

With no config, the spine is built from `content_dir`'s own directory
structure: every `.typ` file, ordered alphabetically per directory level. A
subdirectory with a landing file (`index.typ`, or `<dirname>.typ`) gets its own
clickable page; without one it becomes a non-clickable group titled from the
directory name (a leading numeric prefix like `01-intro/` orders it but is
stripped from the title only — the raw name, prefix included, stays in the
handle).

Two knobs reshape this, both under a global `[spine]` table or a per-format
`[pdf.spine]`/`[html.spine]`/`[epub.spine]` override (per-format overrides are
field-by-field: an unset field still falls back to the global `[spine]`):

```toml
[spine]
exclude = ["drafts/**"]        # globs relative to content_dir, omitted from every format

[[spine.section]]
name    = "chapters"           # virtual directory, no files moved on disk
include = ["ch-*.typ"]         # matched files get handle `chapters:<stem>`
```

`[[spine.section]]` nests via `[[spine.section.section]]`. When `include` lists
several globs, matches are gathered in glob order (lexicographic within one
glob) — list globs in the order you want when you need explicit control.

```toml
[epub.spine]
title = "My Book"
```

PDF combines its spine into a single document by default, rewriting
cross-`.typ` links as internal section references; HTML and EPUB always
produce one output per vertebra. There is no `vertebrae` key any more — order
comes from file/directory naming, `exclude`, and `[[spine.section]]`.

## Assets

Global assets (copied for every format), paths relative to project root:

```toml
copy = ["assets/**/*.png", "fonts/**"]
```

Format-specific assets are merged with global:

```toml
[html.assets]
copy = ["images/**"]
```

Directory hierarchy is preserved in the output.

## Custom JS/CSS for HTML

Single bundle (note: `css_stylesheet` is singular here):

```toml
[html.assets]
css_stylesheet = "./style.css"
js_scripts     = "./index.js"
```

Multiple bundles via array-of-tables (note: `css_stylesheets` plural here):

```toml
[[html.assets]]
dest             = "tooltip"
js_scripts       = "tooltip/index.js"
css_stylesheets  = "tooltip/index.css"

[[html.assets]]
dest        = "annotations"
js_scripts  = "annotations/index.js"
```

Files land in `dest/` under the HTML output (or HTML root if `dest` is omitted).

Gotcha: a custom `style.css` **replaces** the default styles entirely — there
is no merge. Copy the defaults from the rheo repo if you want to extend rather
than override.

## rheo-* variables

Any top-level `#let rheo-<key> = <value>` in a vertebra is harvested at
compile time and exposed to plugins with the `rheo-` prefix stripped (so
`rheo-feed-title` is read as `feed-title`). The right-hand side **must** be a
string or boolean literal — any other value is a compile error. Bindings nested
inside closures or code blocks are not file-scope and are ignored.

The Atom feed variables below (`rheo-feed-title`, `rheo-feed-updated`,
`rheo-feed-exclude`) are an instance of this convention.

## Atom feed (HTML)

Set `feed_base_url` under `[html]` to enable an Atom 1.0 feed:

```toml
[html]
feed_base_url = "https://example.com"
feed_author   = "Jane Doe"            # optional; default "Rheo"
```

Without `feed_base_url`, no feed is emitted. When set, the HTML build writes
`build/html/feed.xml` with one `<entry>` per spine vertebra by default, and
injects a `<link rel="alternate" type="application/atom+xml">` autodiscovery tag
into every page's `<head>`.

`feed_author` is an optional string that sets the feed-level `atom:author`
(`<author><name>…</name></author>`). It defaults to `"Rheo"` when absent;
XML-special characters are escaped automatically.

Per-entry values are top-level `#let` bindings in the vertebra:

```typst
#let rheo-feed-title   = "My first post"
#let rheo-feed-updated = "2026-01-15T00:00:00Z"
#let rheo-feed-exclude = true
```

Every vertebra appears in the feed by default. All three variables are optional:

- `rheo-feed-title` — overrides the entry title; defaults to the document title
  from `#set document(title: ...)`.
- `rheo-feed-updated` — overrides the entry timestamp (RFC 3339); defaults to the
  document date from `#set document(date: ...)`, then the source file's mtime.
- `rheo-feed-exclude` — the boolean `true` omits this vertebra from the feed (its
  page is still built). Useful for cover/index pages.

Each entry's `<content>` is chosen from the page, first match wins:

1. the first `<main>` element;
2. else the first element with class `rheo-feed-content`;
3. else the whole `<body>`.

To keep site chrome (header, footer, nav) out of feed entries, wrap the article
in `<main>` and keep the chrome outside it:

```typst
#show: doc => {
  site-header()
  html.elem("main", doc)   // ← only this becomes the feed entry
  site-footer()
}
```

With no `<main>` or `rheo-feed-content` marker, the full body is used.

## Relative linking between `.typ` files

```typst
#link("./another-section.typ")[See another section]
```

Rheo rewrites these per format:
- HTML → `<a href="another-section.html">`
- EPUB → internal reference
- PDF → an internal section ref (the spine is combined into one document)

This is what makes the same source tree work as a static site, an EPUB, and a
linked PDF.

## Packages

Typst Universe packages can ship web assets. Import normally:

```typst
#import "@preview/rheo-tooltip:0.1.0": tooltip
```

Rheo reads `[tool.rheo.html]` from the package's own `typst.toml` and pulls in
its `js_scripts`, `css_stylesheets`, and `copy` entries automatically. Paths
there resolve relative to the package's location in the Typst cache.

## Slides (`@rheo/slides`)

The `@rheo/slides` package compiles a single `.typ` file to both a RevealJS HTML presentation and a printable PDF script simultaneously.

### Import

```typst
#import "@rheo/slides:0.1.0": template, slide
```

### Defining slides

```typst
#slide(title: [Introduction])[
  Content here. Any Typst content: lists, figures, math, citations.
]
```

`title` is optional. Omit for a heading-free slide.

### Template

```typst
#show: template.with(
  theme: "white",
  transition: "slide",
  first-slide: [
    = My Presentation

    Author Name
  ],
)
```

- `theme` — any built-in RevealJS theme name
- `transition` — `none`, `fade`, `slide`, `convex`, `concave`, `zoom`
- `first-slide` — arbitrary Typst content; renders as the opening slide

### Spine config

A single-file project needs no spine config at all — the directory-scan
default already includes `slides.typ`. Give the PDF a title, if wanted:

```toml
[pdf.spine]
title = "My Presentation"
```

### PDF output

Each `slide` renders as a headed section on standard paper. `first-slide` becomes a title page. Suitable as a printed script or handout.

### Customising RevealJS CSS

Attach a project CSS file via `[[html.assets]]`:

```toml
[[html.assets]]
css_stylesheet = "style.css"
```

Your stylesheet loads after the package base styles. Key RevealJS CSS variables:

- `--r-main-color` — foreground
- `--r-background-color` — background
- `--r-main-font-size` — base font size
- `--r-heading-color` — headings
- `--r-link-color` — links

Common overrides:

```css
/* Title slide heading colour */
.reveal .slides section:first-child h2 { color: #e7ad52; }

/* Font sizes */
.reveal .slides > section  { font-size: 0.8em; }
.reveal .slides blockquote { font-size: 0.9em; }
.reveal .slides figure     { font-size: 1.5em; }
.reveal .slides figcaption { font-size: 0.4em; }

/* Theme-agnostic border using CSS variables */
.reveal .slides figcaption {
  border-top: 1px solid color-mix(in srgb, var(--r-main-color) 25%, var(--r-background-color));
}
```

## When to hand off to typst-author

Raw Typst markup — show rules, `target()`, figures, math, packages — belongs
to `typst-author`. This skill stays focused on the rheo-level glue: project
layout, `rheo.toml`, spines, assets, and the CLI.

## Reference

- Docs: <https://rheo.ohrg.org/>
- Typst docs: <https://typst.app/docs/>
