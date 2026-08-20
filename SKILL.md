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
version = "0.5.1"
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

Gotcha: a custom `css_stylesheet` **replaces** the default styles entirely —
there is no merge. With no user CSS, rheo emits its built-in default as a linked
asset, `rheo-default.css`, in the HTML output; the moment you set your own
`css_stylesheet` that file is no longer emitted. To extend rather than override,
start from `rheo-default.css` and layer your rules on top. Asset `<link>` hrefs
are depth-relative, so nested pages resolve root-level assets correctly.

## Footnotes (HTML/EPUB)

By default the footnote counter **resets per page** in HTML and EPUB bundle
output — each page starts its footnotes at 1 rather than continuing across the
whole spine. Set `reset_footnotes = false` under `[html]` or `[epub]` to keep
continuous numbering across pages:

```toml
[html]
reset_footnotes = false   # continuous across pages (default true = reset per page)
```

Per-format, so HTML and EPUB can differ. PDF is unaffected.

## Feeds (`@rheo/feeds`)

Feeds are configured in Typst, not `rheo.toml`. `@rheo/feeds` emits Atom 1.0,
RSS 2.0, and JSON Feed 1.1 from one config — the retired Rust generator did
Atom only. Requires rheo >= 0.6.0 (the package asserts this itself and fails
the build below that floor).

```typst
#import "@rheo/feeds:0.1.0": feed, configure, spine

#configure(feeds: (
  feed(
    title: "My Site",
    base-url: "https://example.com",
    sources: (spine(),),
  ),
))
```

Call `configure` once, from any vertebra. Every spine vertebra becomes a
candidate entry in `feed.xml` by default.

### Multiple feeds, multiple formats

One `configure(...)` call can register several `feed(...)`s, each with its own
`path` and, optionally, `format` (`"atom"` default, `"rss"`, `"json"`):

```typst
#let posts = spine(filter: e => e.handle.starts-with("posts:"))

#configure(feeds: (
  feed(path: "feed.xml", title: "My Site", base-url: "https://example.com", sources: (posts,)),
  feed(path: "rss.xml",  title: "My Site", base-url: "https://example.com", sources: (posts,), format: "rss"),
))
```

Every page's `<head>` gets one autodiscovery `<link>` per feed.

### Sources

A source is a plain function `cfg => (entries)`. Two are built in:

- `spine(filter:, select:)` — every spine vertebra by default; `filter` is a
  predicate over `(handle, path, title)`, e.g. `spine(filter: e =>
  e.handle.starts-with("posts:"))` to narrow to one directory.
- `items(filter:, label-name:)` — entries any package or page contributes via
  a `#metadata((...)) <feeds:item>` beacon (or the `item(...)` helper that
  writes one for you). Reach for this only when there's no accessor to call
  directly.

`@rheo/rookery` notes are syndicated the direct way first: a hand-written
source calls `ideas(tags:)` and reshapes its rows onto the entry shape (see
the package readme's "Sourcing from another package" for the worked recipe).
`#show: rookery.with(syndicate: true)` is the fallback — it makes each minted
note page emit a `<feeds:item>` beacon for `items()` to pick up, for sources
with no accessor to call.

`content` (default `"html"`) splices the entry's own page into `<content>`;
set it `none` when the entry's page isn't a compiled vertebra (a rookery note
is minted, not compiled) or when the feed's `format` is `"json"` (required
there — JSON Feed has no way to carry rheo's spliced-in page HTML).

### Behaviour to get right

- **`title` is required, with no fallback chain.** The retired generator fell
  back from an explicit title to the HTML spine's own title to the project's
  directory name; `feed(...)` panics on an empty title instead.
- **An entry with no date is dropped, not defaulted.** Atom requires
  `<updated>`, and Typst can't stat a compiled output file's mtime, so a
  `spine()` entry with no `#set document(date: ...)` — and nothing else
  supplying a date — silently never becomes a candidate. This is also how you
  exclude a cover or index page from a feed: leave it undated.
- **Never date a page with `datetime.today()`.** It resolves to whatever day
  the build runs, so a syndicated page's timestamp changes on every rebuild.
  Write a literal `datetime(year: ..., month: ..., day: ...)`.

### Removed in 0.6.0: the old `[html]` feed config

`[html] feed_base_url`, `feed_author`, `feed_title`, `[[html.feed_include]]`,
and the per-vertebra `#let rheo-feed-title` / `rheo-feed-updated` /
`rheo-feed-exclude` bindings are deleted from the engine outright, not
deprecated. A build that still sets one now WARNS, naming the key (and, for
the `.typ` bindings, the file and line); `rheo migrate` reports the same
findings but does not rewrite them, since the old keys don't map onto
`@rheo/feeds` config one-to-one. See the package readme's "Migrating from the
retired Rust feed generator" for the full mapping.

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
