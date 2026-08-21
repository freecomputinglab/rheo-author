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
version = "0.6.0"
content_dir = "./"
build_dir   = "build"
formats     = ["pdf", "html", "epub"]

[epub.spine]
title = "<project dir name>"
```

Restrict outputs by trimming `formats`, e.g. `formats = ["html", "epub"]`.

`version` must match the installed rheo. A mismatch WARNS (does not fail
the build) naming both versions; run `rheo migrate` to update `rheo.toml`
in place (add `--apply` to write the change — without it, `rheo migrate`
only reports what it would do).

### Path resolution (load-bearing gotcha)

Once `content_dir` is set, `build_dir` **and all spine globs** resolve relative
to `content_dir`, not the project root. Asset `copy` paths stay relative to the
project root. Mix these up and files land in the wrong place.

## Spines — directory-scan default, exclude, include, sections

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

`[[spine.section]]` nests via `[[spine.section.section]]`. When *this* `include`
(the section's own field) lists several globs, matches are gathered in glob
order (lexicographic within one glob) — list globs in the order you want when
you need explicit control.

`[spine] include` is a **different key with the same name**, set directly on
`[spine]`/`[<format>.spine]` rather than inside a `[[spine.section]]` block. It
reorders the scan in place instead of grouping into a virtual directory — no
new handle prefix, no path change, just a different order — and drops any file
none of its patterns match (so it also replaces a same-scope `exclude`):

```toml
[spine]
include = ["index.typ", "install.typ", "ideas.typ", "flights.typ"]
```

`include` and `section` are rejected together on one table — pick one; an
`include` pattern matching no file is also a build error. First version
reorders flat, top-level files only, not within a nested content directory.

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

Multiple bundles via array-of-tables (`css_stylesheet` stays singular in
every block — there is no plural form):

```toml
[[html.assets]]
dest            = "tooltip"
js_scripts      = "tooltip/index.js"
css_stylesheet  = "tooltip/index.css"

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

## Cross-file references

Three ways to reference across `.typ` files: relative path links, `<handle>`
anchors, and `rheo-context()` for reading another vertebra's state from Typst.

### Relative path links

```typst
#link("./another-section.typ")[See another section]
```

Rheo rewrites these per format:
- HTML → `<a href="another-section.html">`
- EPUB → internal reference
- PDF → an internal section ref (the spine is combined into one document)

This is what makes the same source tree work as a static site, an EPUB, and a
linked PDF.

### Labels and handle anchors

Rheo does NOT rewrite or prefix authored Typst labels — they stay exactly as
Typst defines them, in ONE flat global namespace across the whole spine. Two
vertebrae defining the same label collide as an ordinary Typst duplicate-label
error, same as within a single file.

Separately, rheo synthesizes a `<handle>` anchor per vertebra (plus a
`<handle.typ>` escape alias) so one vertebra can reference another by handle:

```typst
#link(<chapters:intro>)[nested page]
@chapters:intro
```

These anchors are labeled `#figure` elements (`#metadata` and bare labels
aren't cross-document-referenceable in Typst 0.15), hidden from render, with
`@handle` shown as a link carrying the target vertebra's title. If a
user-authored label already claims the canonical handle name, rheo silently
skips injecting it — the vertebra stays reachable via its `<handle.typ>`
escape form. An escape form colliding with anything else IS a hard build
error naming the file and label.

The whole spine compiles as ONE Typst document, so `query()`/
`state(...).final()` see every vertebra — this is why handle references
resolve with no post-processing step.

A link to an AUTHORED label in another vertebra (not a `<handle>` anchor)
resolves correctly too, even though rheo's own link-rewrite rule never
touches it — that rule only rewrites `rheo-handle` figures. Typst's own
bundle-target link machinery (not rheo) computes the right cross-page href
and fragment for any label, wherever it's defined in the spine. Reach for a
`<handle>` anchor anyway when you want the link's own text to be the target's
title automatically (`@chapters:intro`); reach for a plain label when you
already have your own text to show.

### `rheo-context()`

Rheo injects a zero-arg function into every vertebra:

```typst
#rheo-context().handle           // this file's handle, e.g. "chapters:intro"
#rheo-context().spine-flat.len() // every clickable vertebra, flat, pre-order
```

Fields:
- `handle` — this vertebra's own `:`-separated handle. The only per-file
  field; everything else below is shared across the whole spine.
- `spine` — the structured spine tree (`title`/`handle`/`path`/`children` per
  node; a group node has `handle`/`path` `none`). `title` here is always
  **path-derived** — never a vertebra's own `#set document(title: ...)`.
- `spine-flat` — the flat pre-order list of clickable vertebrae, same
  path-derived-title caveat.
- `metadata-of` — a closure, `(handle) => dict`, reading another vertebra's
  resolved `#set document(...)` values live (see below).
- `rheo-version` — the compiling rheo's own semver string (e.g. `"0.6.0"`),
  always present.
- `target` / `ext` — output format name / file extension. Present for HTML
  and EPUB; both ABSENT for PDF. Check `"target" in rheo-context()` before
  reading either.

None of these need `#context` EXCEPT `metadata-of` — they read straight off
`sys.inputs`.

**Passing it to a package.** A package can't see a vertebra's local
`rheo-context()` implicitly — Typst functions capture their definition scope,
not the call site — so hand it in explicitly:

```typst
#import "@rheo/somepackage": template
#show: template.with(ctx: rheo-context())
```

**Detecting a rheo build.** `sys.inputs.rheo-context` (a plain dict, not a
function) is the file-independent half of the above — no `handle`, everything
else. Use it to turn a native `typst compile` into a friendly error:

```typst
#let ctx = if "rheo-context" in sys.inputs { rheo-context() } else {
  panic("Compile this with Rheo — https://rheo.ohrg.org")
}
```

**Detecting rheo's version — three states, not two:**
- `"rheo-context" not in sys.inputs` — no rheo at all, plain `typst compile`.
  Not an error; this is a package's own feature-detect path.
- `"rheo-context" in sys.inputs` but no `rheo-version` key — a rheo older
  than the release that added the key.
- `rheo-version` present — compare it and decide.

#### `metadata-of`: reading another vertebra's real metadata

`metadata-of` reads a vertebra's *resolved* `#set document(...)` values —
title, author, description, keywords, date — off a per-vertebra beacon, live,
after Typst has resolved the whole style chain. It's a dict field, not a
method, so call it with the extra parens, and it **requires `#context`** (it
calls `query()` internally) — the one field of `rheo-context()` that does,
since everything else reads a plain `sys.inputs` dict:

```typst
#context [
  #let m = (rheo-context().metadata-of)("chapters:intro")
  #m.at("title", default: [Untitled])
]
```

`title`/`description` come back as real Typst content (not flattened
strings) — a nav built from this can render a formatted title, not just
plain text. `author`/`keywords` are always arrays; `date` is a real
`datetime`. A key the vertebra never set is omitted, not `none`. Returns
`(:)` for a handle with no beacon.

Most authoring forms resolve correctly — a title set via an imported `#show:`
template, a non-literal expression, or multiple `#set document(...)` rules —
because the beacon reads the live, fully-resolved value instead of
re-parsing source text.

**`rheo-context().spine`/`spine-flat` titles do NOT use this** — they stay
path-derived always. Read a vertebra's real authored title via `metadata-of`
instead of `spine-flat[...].title`.

**Combined PDF has no per-vertebra metadata.** PDF's default layout puts
every vertebra into one shared `#document(...)` block, so there's no
well-defined "this vertebra's own metadata" — `metadata-of` returns `(:)` for
every handle there. By design, not a bug.

**Gotcha: a title set inside a bounded `#{ }`/`#[ ]` block is invisible to
`metadata-of`.** The vertebra's own compiled `<title>`/PDF `/Title` is
unaffected — Typst's document-info collection is unscoped — but the beacon
reads `document.title` via `#context`, appended once after the vertebra's own
body, and that read can't see a `set` whose bounded block already closed
earlier in the same file. So `metadata-of` and any `@handle` reference to
that vertebra silently fall back to the path-derived title instead. A
`#show: template` rule is unaffected (it has no closing brace of its own —
it applies through the end of the enclosing block, where the beacon lives
too). Give a title at the vertebra's top level, or via `#show:`, if it needs
to be visible to another vertebra's metadata read or a handle anchor.

**Reserved label prefix:** the beacon is labelled `<rheo-meta:handle>`. An
authored label starting with `rheo-meta:` is a hard build error naming the
file and label.

## Packages

Typst Universe packages can ship web assets. Import normally:

```typst
#import "@preview/rheo-tooltip:0.1.0": tooltip
```

Rheo reads `[tool.rheo.html]` from the package's own `typst.toml` and pulls in
its `js_scripts`, `css_stylesheet`, and `copy` entries automatically. Paths
there resolve relative to the package's location in the Typst cache.

### `[tool.rheo] min_version`

A package declares the oldest rheo it works with:

```toml
[tool.rheo]
min_version = "0.6.0"
```

A plain floor, no ranges. A running rheo older than this FAILS the build,
naming the offending package. Omitting the key means no check at all. Only
enforced by rheo versions that have the check themselves — a package that
must fail cleanly even on an ancient rheo should also `assert` a floor in its
own Typst using `rheo-context().rheo-version` (see `rheo-context()` above),
since the manifest check can't protect against a rheo too old to read it.

Gotcha: `[tool.rheo]` must be the LAST table in the file. A
`[tool.rheo.<format>]` subtable declared after it would capture `min_version`
into that subtable instead.

## Notes (`@rheo/rookery`)

Zettelkasten-style atomic notes — interlinked, transcludable, with their own
minted pages under rheo.

```typst
#import "@rheo/rookery:0.4.0": idea, window

#idea("etal")[A pinned note — id is always `idea:etal`.]
#window("etal")   // transcludes it inline, foldable
@idea:etal        // terse ref — renders the note's title, linked
```

No `ctx:` parameter and no template required to use `idea`/`window` — rheo
needs nothing extra. `#show: rookery.with(...)` (optional) wires up
`@idea:etal` to render the note's title instead of a bare figure number, plus
prefix/theme/bibliography config; see the package readme for the full option
set. `#note(...)`/`#todo(...)` are sugar over `idea` that prepend a `note`/
`todo` tag.

Note ids are FLAT and globally unique (`idea:name`, no per-file prefix) — a
note keeps its id when it moves between files, and a duplicate id is a build
error naming it. `#ideas()` (inside `#context`) hands back every note as
data — `id`/`name`/`title`/`text`/`tags`/`body`/`href`/`page`/`minted`/
`updated` per entry — the seam for a custom index, feed, or search over the
corpus; see "Sourcing from another package" in the Feeds section above for
the worked recipe feeding `@rheo/feeds` from it.

Full docs: `rheo-packages/rookery/<version>/readme.md`; a worked multi-page
example lives at `rookery.ohrg.org`.

## Marrow (`.marrow.typ`)

`content/.marrow.typ` is inlined at the Typst bundle root, outside every
page. It is NOT a vertebra: no `.marrow.html` output, never in the spine,
never in nav. Default filename is `.marrow.typ`; override with a top-level
`marrow = "..."` key in `rheo.toml`, resolved against `content_dir`.

It is the only place `#document(...)` and `#asset(...)` are legal — how
extra output files get minted:

```typst
#document("extra/hello.html", format: "html", title: [Extra])[Hello from the bundle root.]
#asset("extra/hello.txt", "root-level asset")
```

Call either from inside a vertebra and the build fails: `setting the
document format is only supported in the bundle target`.

Per-page formats only (`html`, `epub`). PDF combines its spine into one
document, so marrow is skipped there entirely.

### One page per registered item

```typst
#context {
  for n in state("notes", ()).final() {
    document("notes/" + n.name + ".html", format: "html", title: [Note])[#n.body]
  }
}
```

A top-level `#context` reading `state(...).final()` and calling
`document()`/`asset()` in a loop is how marrow turns arbitrary registered
data into one output per item — the same shape `@rheo/feeds`'s own
`.marrow.typ` uses to mint every configured feed.

### Package marrow

A package can ship its own `.marrow.typ`; importing the package is enough
for it to run, alongside the project's own marrow, not instead of it. Turn
it off per format with `auto_detect_packages = false` (e.g. under `[html]`).

### Gotchas

- Paths inside marrow resolve against the **project root**, not
  `content_dir`.
- Typst can't list a directory — marrow finds things through `state`,
  labels, or `sys.inputs.rheo-context`, never by scanning `content/`.
- A `#show` rule inside marrow affects only what marrow itself mints, not
  existing vertebrae.

## Bundle-output primitives

Most authors never write these directly — a package such as `@rheo/feeds`
uses them so you don't have to. Reach for them only when building
something no package already covers (want an Atom/RSS/JSON feed? use
`@rheo/feeds` — see "Feeds" above, not this section).

### Transclusion: `<rheo-content>`

A marrow-minted asset can embed another page's compiled HTML with a
placeholder, resolved after compilation (once real page HTML exists):

```text
<rheo-content page="notes/etal.html" select="main" as="escaped"/>
```

- `page` (required) — a compiled page's plugin-output-relative path.
- `select` (optional) — a bare tag name (`main`) or leading-dot class
  (`.rheo-content`); default cascade is `<main>` → `.rheo-content` →
  `.rheo-feed-content` → whole `<body>`.
- `as` (optional) — `escaped` (default; entity-escaped, for `<content
  type="html">`), `raw` (verbatim, for `<content type="xhtml">`), or
  `json` (escaped as a JSON string body, for a JSON Feed's
  `content_html`).

No Typst function does this: marrow runs *inside* the Typst compile, before
any page HTML exists, so there is no HTML string to hand back yet.

### Head contributions

Two routes — `#set document(...)` alone can't reach `<head>` otherwise:

- `<rheo-head>` wrapped around content anywhere in ONE page's body — its
  children are hoisted into that page's own `<head>`, wrapper removed:

  ```typst
  #html.elem("rheo-head", html.elem("link", attrs: (rel: "canonical", href: "https://example.com/a.html")))
  ```

- `.rheo/head.html` minted from marrow — an HTML fragment (no
  `<html>`/`<head>`/`<body>` wrapper) appended to EVERY page's `<head>`:

  ```typst
  #asset(
    ".rheo/head.html",
    "<link rel=\"alternate\" type=\"application/atom+xml\" href=\"https://example.com/feed.xml\" title=\"Site Feed\">",
  )
  ```

HTML only. The only EPUB guarantee is that control assets stay out of the
EPUB container — no head-injection behaviour is promised there.

### Control assets: the `.rheo/` prefix

Anything minted under `.rheo/` (e.g. `.rheo/head.html`) is a message from
the bundle to rheo: consumed during compilation, never written to the
build output. An unrecognized `.rheo/*` path is dropped with a warning,
not a build failure.

### End to end

```typst
// content/.marrow.typ
#asset(
  ".rheo/head.html",
  "<link rel=\"alternate\" type=\"application/atom+xml\" href=\"https://example.com/feed.xml\" title=\"Site Feed\">",
)
#asset(
  "feed.xml",
  "<entry><rheo-content page=\"notes/etal.html\" as=\"escaped\"/></entry>",
)
```

Mints `feed.xml` with `notes/etal.html`'s compiled `<main>` spliced in, and
adds an autodiscovery `<link>` to every page's `<head>`.

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
