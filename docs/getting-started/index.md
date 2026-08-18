# Start here

By the end of this tutorial, an imported `index.html` file will be visible and
clickable in Play In Editor. The page will live in a normal UMG Widget
Blueprint, fill a Canvas Panel, and be connected to a Magic UI Component on an
Actor.

This is the complete beginner path. Follow the four pages in order even if you
already know UMG—the important MagicUI connection is split between the Actor
component and the UMG widget.

![The parts of a MagicUI screen](../assets/diagrams/screen-pipeline.svg)

## What you need

- Unreal Engine **5.8**.
- A Win64 or macOS arm64 project and a compatible packaged MagicUI plugin. See
  [Platforms and limitations](../reference/platforms-and-limitations.md) for
  platform-specific behavior.
- Permission to add a project plugin and restart Unreal Editor.
- Any plain-text web editor, such as Visual Studio Code, Rider, WebStorm, or
  Notepad++.
- A Blueprint project or a C++ project; the tutorial itself uses Blueprints.

!!! warning "Use the packaged plugin"

    Install the complete, already-built Unreal plugin package. Its root should
    contain `MagicUI.uplugin`; keep the supplied folders together.

## The four stages

<div class="feature-grid" markdown>

<div class="feature-card" markdown>

### 1. [Install and enable](install-and-enable.md)

Place `MagicUI` beneath the project's `Plugins` directory, enable it, restart,
and confirm Unreal loaded both runtime and editor modules.

</div>

<div class="feature-card" markdown>

### 2. [Create the HTML](create-html.md)

Create a self-contained first page with a visible button and a safe
JavaScript-to-Unreal message.

</div>

<div class="feature-card" markdown>

### 3. [Import the HTML asset](import-html-asset.md)

Turn `index.html` into a **MagicUI Web File**. This imported Unreal asset is the
source used by the component and the cook.

</div>

<div class="feature-card" markdown>

### 4. [Render the screen](first-screen.md)

Build the Actor, Canvas Panel, Magic UI widget, runtime source connection, and
viewport graph.

</div>

</div>

## Keep this mental model nearby

The **component** is the browser. The **widget** is the screen and input bridge.
The **web asset** is the local page package. A Canvas Panel is simply where the
screen widget is laid out.

```text
MagicUI Web File  →  Magic UI Component  →  rendered texture
                              ↓
Canvas Panel      ←     Magic UI widget  ←  mouse / key input
```

If the page is blank, inspect that chain from left to right. Most first-time
setup issues are either an unassigned HTML Asset or a UMG widget whose Source
Component was never set.

## What this tutorial deliberately does not do

- It does not load a website. MagicUI is local-bundle-only.
- It does not call `Initialize UI` or `Load HTML From Asset` in BeginPlay. An
  assigned HTML Asset is loaded automatically by the component.
- It does not enable **Resize View To Widget**. A full-screen component already
  uses **Auto Resize To Screen**, and that setting owns the view size.
- It does not use the named event helper yet. The first button uses the raw
  bridge so you can verify the smallest possible setup.

Continue with [Install and enable](install-and-enable.md).
