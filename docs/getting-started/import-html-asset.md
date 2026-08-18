# 3. Import the HTML asset

MagicUI's recommended content path starts with an Unreal asset. Importing the
entry HTML creates a **MagicUI Web File** that remembers the source file and
stores a cookable snapshot of its supported sibling resources.

## Import `index.html`

1. Return to Unreal Editor and make sure Play In Editor is stopped.
2. In the Content Browser, create `/Game/MagicUI/Web`.
3. Open File Explorer or Finder at `WebSource/GettingStarted`.
4. Drag `index.html` into `/Game/MagicUI/Web`.
5. Wait for the import to finish.
6. Rename the imported asset to `MWA_GettingStarted` if you use Unreal naming
   prefixes.

The asset type should read **MagicUI Web File**. It is not a generic File Media
Source, Text asset, or Unreal Web Browser asset.

![Imported web source and cookable snapshot](../assets/diagrams/web-asset-lifecycle.svg)

<div class="step-result">The Content Browser contains <strong>MWA_GettingStarted</strong>, and its asset type is <strong>MagicUI Web File</strong>.</div>

## Inspect the linked source

Double-click the asset. Its custom Details view shows:

| Field or action | What it means |
|---|---|
| **Source File** | The external `index.html` linked to this Unreal asset. |
| **Status** | `Exists` when Unreal can still find that source file. |
| **External Editor** | The application used by **Open in External Editor**. |
| **Choose Editor…** | Select VS Code, Rider, WebStorm, or another editor. |
| **Use OS Default** | Return to the operating system's file association. |
| **Open in External Editor** | Edit the authoritative file on disk. |
| **Reveal in File Explorer** | Locate the source file or its directory. |
| **Refresh Packaged Copy** | Update the embedded last-good snapshot used for cooking. |

The asset also tracks its source filename, type, and bundled file count. The
first all-in-one example normally has one bundled file.

## Understand what was imported

For an HTML entry asset, MagicUI recursively collects supported files from the
HTML file's entire directory—not only files discovered by parsing `<link>` and
`<script>` tags. That is why each UI should have a clean source folder.

```text
WebSource/GettingStarted/   ← one page-specific source root
├── index.html              ← imported entry point
├── css/main.css            ← bundled automatically
├── js/app.js               ← bundled automatically
└── images/icon.png         ← bundled automatically
```

The complete extension and limit table is in [Supported content](../reference/supported-content.md).

!!! warning "The HTML asset matters in packaged games"

    `HTML File Path` and loose disk paths work only in Editor/PIE. A cooked or
    packaged build needs the imported HTML **MagicUI Web File**. Do not defer
    this import until packaging day.

## Edit-and-test loop

Editor/PIE follows the linked source, but each play/view session receives an
immutable snapshot. Use this loop:

1. Stop PIE and make sure all live MagicUI views are gone.
2. Edit and save HTML, CSS, JavaScript, images, or fonts on disk;
3. start PIE again; and
4. use **Refresh Packaged Copy**, then save the Unreal asset before a cook or
   source-control handoff.

`Reload` reloads the current session's page. It cannot replace that session's
frozen source bundle with a newly edited folder digest. Stop and start PIE for
source changes.

## If the linked file moves

Use the Content Browser's standard **Reimport With New File** action and select
the new entry file. Confirm the custom Details panel reports **Status: Exists**,
refresh the packaged copy, and save the asset.

Next, [build the Actor, Canvas Panel, and Magic UI widget](first-screen.md).
