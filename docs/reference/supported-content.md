# Supported web content

MagicUI renders **local HTML bundles** inside the Unreal plugin. The intended
entry point is an imported **MagicUI Web File** whose type is HTML. HTML, CSS,
JavaScript, images, fonts, data files, and WebAssembly can travel with that
entry page when they use a supported extension.

!!! warning "Local-bundle-only runtime"

    This runtime profile cannot browse the web. Top-level and redirected
    navigation is restricted to immutable, internal `magic://bundle/...` URLs.
    `http:`, `https:`, `file:`, `data:`, and arbitrary schemes are rejected,
    and persistent cache/storage directories are disabled. There is no MagicUI
    project setting that turns remote networking on.

## Files you can import into Unreal

The editor importer recognizes these files as standalone **MagicUI Web File**
assets:

| Import extension | Asset type | Can be an HTML Asset entry page? |
| --- | --- | --- |
| `.html`, `.htm` | HTML | **Yes** |
| `.css` | CSS | No |
| `.js`, `.mjs` | JavaScript | No |

Only an HTML asset passes **Load HTML From Asset**. Import CSS or JavaScript on
its own only when having a separate Content Browser record is useful to your
workflow; ordinary page dependencies should live beside the HTML source and
are collected into its bundle automatically.

For the beginner import flow, start at
[Import the HTML asset](../getting-started/import-html-asset.md).

## Files an HTML import can bundle

When the imported entry is HTML, MagicUI walks the entry file's **entire source
directory recursively** and caches every file with one of these extensions:

| Category | Extensions |
| --- | --- |
| Documents and styles | `.html`, `.htm`, `.css` |
| Scripts and structured/text data | `.js`, `.mjs`, `.json`, `.xml`, `.txt` |
| Vector and raster images | `.svg`, `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.ico` |
| Web fonts | `.woff`, `.woff2`, `.ttf`, `.otf` |
| WebAssembly | `.wasm` |

The importer does not parse an HTML dependency graph. A supported file beneath
the folder is bundled even when the entry page never references it; an
unsupported file is skipped even when the page does reference it.

```text
WebSource/MyMenu/                ← import root is this whole directory
├── index.html                  ← drag this into Content Browser
├── styles/menu.css             ← bundled
├── scripts/menu.js             ← bundled
├── data/items.json             ← bundled
├── images/logo.webp            ← bundled
├── fonts/inter.woff2           ← bundled
├── effects/layout.wasm         ← bundled
├── video/intro.mp4             ← NOT bundled
└── notes/design.psd            ← NOT bundled
```

This is why each UI should have a small, dedicated source directory. Importing
an `index.html` from a broad project folder can collect unrelated resources and
hit bundle limits.

### Audio and video

Audio/video extensions such as `.mp3`, `.wav`, `.ogg`, `.mp4`, and `.webm` are
not in the Unreal bundle allowlist. They will not be packaged as MagicUI web
resources. Remote media is unavailable as well because the runtime is
local-bundle-only. Do not design a MagicUI screen around bundled audio or
video.

## Write bundle-relative URLs

Reference sibling content with paths relative to the document or a local
bundle root:

```html
<link rel="stylesheet" href="styles/menu.css">
<img src="images/logo.webp" alt="Studio logo">
<script src="scripts/menu.js" defer></script>
```

Avoid source-machine paths and remote URLs:

```html
<!-- Wrong for a cooked MagicUI bundle -->
<script src="C:\Users\Me\WebSource\menu.js"></script>
<img src="file:///C:/WebSource/images/logo.png">
<script src="https://cdn.example.com/menu.js"></script>
```

MagicUI assigns an internal URL such as:

```text
magic://bundle/asset_<identity>_<digest>/index.html
```

Treat that URL as runtime-owned. Do not hard-code the generated mount identity
in authored HTML or Blueprints.

## Safe filename rules

The runtime validates every bundled relative path before mounting it. Use
portable ASCII names for all files and folders.

A valid path:

- is relative, with `/` separators;
- has no empty segment, `.` segment, or `..` traversal;
- contains printable ASCII only;
- is no more than 4,096 UTF-8 bytes total;
- has no segment longer than 255 UTF-8 bytes;
- has at most 64 path segments; and
- has no case-insensitive collision with another bundled path.

The following are rejected:

- a leading/trailing slash, doubled `//`, backslash, or colon;
- encoded `/`, encoded `\`, or encoded NUL;
- control characters or non-ASCII characters in a filename;
- `<`, `>`, `"`, `|`, `?`, or `*`;
- a segment ending in a period or space;
- `CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`, or `LPT1`–`LPT9`, including those
  names before a file extension; and
- names that differ only by letter case, such as `Logo.png` and `logo.png`.

Spaces inside a filename are accepted and URL-encoded, but simple lowercase
names such as `images/player-avatar.png` are easier to move between platforms
and debug.

!!! example "Recommended naming"

    ```text
    ui-main/index.html
    ui-main/styles/main.css
    ui-main/scripts/player-status.js
    ui-main/images/player-avatar.png
    ```

## Bundle safety limits

The runtime uses bounded memory and queues. A single prepared HTML bundle is
subject to these user-visible limits:

| Limit | Maximum |
| --- | ---: |
| Files in one bundle | 4,096 |
| Total bytes in one bundle | 64 MiB |
| Bytes in one resource response | 1 MiB |
| Relative path | 4,096 UTF-8 bytes |
| One path segment | 255 UTF-8 bytes |
| Directory depth | 64 segments |
| Encoded internal URL | 8,192 UTF-8 bytes |

The editor's import cache skips any resource that would exceed its 64 MiB
safety budget and logs the skipped file. Runtime validation is stricter: an
oversized individual file, unsafe name, case-insensitive collision, excessive
file count, or oversized bundle makes preparation fail rather than silently
serving a partial runtime bundle.

There are additional process-wide aggregate bounds for the number of mounts,
files, filesystem nodes, keys, and prepared bytes. Ordinary projects should
not approach them. If a project creates hundreds of separate UI bundles or
continually generates new Editor snapshots, restart the editor and consolidate
the content instead of treating the process as an unbounded web cache.

## Imported asset fields

A **MagicUI Web File** exposes these read-only values to Blueprint:

| Field | What it tells you |
| --- | --- |
| **Web Asset Type** | HTML, CSS, or JavaScript, based on the entry extension. |
| **Source File Name** | Main filename, including its extension. |
| **Source Text** | Cached UTF-8 text for the main file used by cooked builds. |
| **Bundled File Count** | Number of files currently cached for cooking. |
| **Is HTML** | Whether this asset can be used as an HTML entry page. |

See the full [Blueprint API](blueprint-api.md#magicui-web-file).

## Linked source in the editor

The `.uasset` remembers the external source path. Select the asset and use its
**Linked Source File** section:

| Control | Use |
| --- | --- |
| **Source File / Status** | Shows the path and whether it still exists. |
| **Choose Editor…** | Selects an external editor for MagicUI web files on this computer. |
| **Use OS Default** | Returns to the operating system file association. |
| **Open in External Editor** | Opens the linked main file for authoring. |
| **Reveal in File Explorer** | Reveals the file, or its containing folder when the file is missing. |
| **Refresh Packaged Copy** | Reads the main file and supported resources again, then updates the embedded cook fallback. |

Normal Unreal **Reimport** and **Reimport With New File** are also supported.
The linked source is authoritative for Editor work; the embedded copy is the
last successfully refreshed fallback for cooking.

## Editor and PIE snapshot behavior

Editor/PIE loose content is copied into an immutable local bundle for a view
session. This protects the runtime from files changing halfway through a page
load, but it also means edits are not hot-reloaded into an active session.

Use this edit loop:

```text
Stop PIE
  → save HTML/CSS/JS/images in the external source folder
  → optionally Refresh Packaged Copy / Reimport
  → start PIE again
```

**Reload** can reload the current immutable snapshot; it cannot introduce a
new content digest after the active bundle session has frozen. If MagicUI says
the session is immutable, stop and restart PIE (or destroy every MagicUI view)
before loading the changed loose snapshot. Restarting the whole editor creates
a completely fresh process generation when accumulated snapshot bounds are the
problem.

Editor path aliases for **HTML File Path** are:

| Path form | Resolution |
| --- | --- |
| `PluginContent:/Web/index.html` | Beneath the MagicUI plugin's Content directory. |
| `ProjectContent:/UI/index.html` | Beneath the Unreal project's Content directory. |
| Relative path | Beneath the MagicUI plugin's Content directory. |
| Absolute path | Used directly in Editor/PIE before snapshotting. |

Loose paths are deliberately disabled in packaged games.

## Cooking and packaged games

Cooked builds use the files embedded in the imported HTML asset, not an
absolute development-machine path.

During cook, MagicUI tries to refresh the embedded copy from the linked source.
If the source is missing or invalid, it preserves the last successful fallback
and logs an error. A cook can therefore contain older content if you ignore the
log. Refresh the asset deliberately, save it, and treat every MagicUI cook error
as actionable.

!!! danger "Stage possible pages before the first view"

    The packaged bundle catalog is fixed when the process runtime starts. Every
    cooked HTML asset that the game may select should already be referenced by
    a loaded Magic UI Component before the first runtime view is created. A
    genuinely new packaged asset selected after that boundary is rejected with
    a message asking you to stage it before the first view or restart Unreal.

The simplest safe setup is one loaded host actor/component for each page that
may be selected, with its imported asset referenced before any view begins.
For project packaging steps, see
[Cook and package web UI](../packaging/cooking-and-packaging.md).

## Inline HTML

**Load HTML From String** remains local and uses MagicUI's built-in inline
bundle as its base. It is suitable for a small self-contained document:

```html
<!doctype html>
<meta charset="utf-8">
<style>body { color: white; background: #111; }</style>
<h1>Inline diagnostic page</h1>
```

It does not create an authored bundle of sibling images, stylesheets, or
scripts. Prefer an imported HTML asset for real interfaces, source control,
relative resources, and cooking.

## What "supported" means

The allowlists above describe what the Unreal plugin imports and serves.
Browser APIs, media codecs, WebAssembly features, and third-party web
frameworks can have additional compatibility constraints. Features that
require remote network access, persistent browser storage, an unsupported
resource extension, or a helper process are outside the current profile. Test
the exact HTML/CSS/JavaScript behavior your UI needs on every target platform
listed in [Platforms and limitations](platforms-and-limitations.md).
