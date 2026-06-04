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
vertebrae = ["**/*.typ"]
title     = "<project dir name>"
```

Restrict outputs by trimming `formats`, e.g. `formats = ["html", "epub"]`.

### Path resolution (load-bearing gotcha)

Once `content_dir` is set, `build_dir` **and all spine globs** resolve relative
to `content_dir`, not the project root. Asset `copy` paths stay relative to the
project root. Mix these up and files land in the wrong place.

## Spines — per-format ordering

Each format has its own spine: `[pdf.spine]`, `[html.spine]`, `[epub.spine]`.

```toml
[epub.spine]
title     = "My Book"
vertebrae = ["intro.typ", "chapters/*.typ", "outro.typ"]
```

Named files preserve order; globs expand lexicographically. The PDF spine
accepts a `merge` attribute that rewrites cross-`.typ` links as internal
section references rather than external links.

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

## Relative linking between `.typ` files

```typst
#link("./another-section.typ")[See another section]
```

Rheo rewrites these per format:
- HTML → `<a href="another-section.html">`
- EPUB → internal reference
- PDF → plain text, or an internal section ref when the spine declares `merge`

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

## When to hand off to typst-author

Raw Typst markup — show rules, `target()`, figures, math, packages — belongs
to `typst-author`. This skill stays focused on the rheo-level glue: project
layout, `rheo.toml`, spines, assets, and the CLI.

## Reference

- Docs: <https://rheo.ohrg.org/>
- Typst docs: <https://typst.app/docs/>
