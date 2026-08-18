# Blueprint API reference

This page lists the Blueprint-facing surface of the **MagicUI Unreal plugin
2.0.0**. It covers the Actor components, UMG widget, imported web asset,
delegates, functions, and enums that Unreal users can see.

## Read this first

MagicUI is asynchronous. A `True` return from a load, message, render-mode, or
event node normally means that the request was accepted into a bounded queue.
It does not mean the browser finished the work.

Use the corresponding event or state check:

```text
load accepted → On Page Loaded / On Load Failed
script queued → On JavaScript Result
frame pending → Has Displayed Frame
mode requested → On Active Render Mode Changed / Get Active Render Mode
event queued  → On Event Error reports later preparation failures
```

An assigned **HTML Asset** is loaded automatically during `BeginPlay`; its load
also initializes the view. Calling **Initialize UI** yourself is not required
for the normal imported-asset workflow, and initialization alone creates a
blank view rather than loading a page.

## Magic UI Component

`Magic UI Component` (`UMagicUIComponent`) is a Blueprint-spawnable Actor
Component. It owns one runtime view, loads local content, receives rendered
frames, and is the endpoint for JavaScript and input.

### Content properties

| Blueprint property | Type / default | Meaning |
| --- | --- | --- |
| **HTML Asset** | `MagicUI Web File`, empty | Imported HTML asset loaded automatically at `BeginPlay`. When assigned, it takes priority over **HTML File Path**. |
| **HTML File Path** | String, `PluginContent:/Web/index.html` | Editor/PIE loose-file source. Loose files are not available in packaged games; use an imported HTML asset there. |

Only an imported asset whose type is **HTML** can be loaded. CSS and JavaScript
assets can be imported for inspection, but cannot be used as a component entry
page. See [Supported content](supported-content.md).

### View properties

| Blueprint property | Type / default | Meaning |
| --- | --- | --- |
| **Auto Resize To Screen** | Boolean, `True` | Continuously follows the game/PIE viewport size and backing scale. For a normal full-screen UMG view, leave this enabled. |
| **Auto Match Viewport DPI** | Boolean, `True` | Uses the viewport's native backing scale. Disable it to use **Manual Device Scale**. |
| **Manual Device Scale** | Float, `1.0` | Fixed device scale when automatic DPI matching is off. Accepted range is `0.25`–`8.0`; the Details slider shows `0.5`–`4.0`. |
| **Transparent** | Boolean, `True` | Requests transparent page output. Your HTML/CSS must also use transparent backgrounds for pixels to remain transparent. |
| **View Width** | Integer, `1280` | Logical CSS viewport width. At least `1`; automatic screen sizing can update it. |
| **View Height** | Integer, `720` | Logical CSS viewport height. At least `1`; automatic screen sizing can update it. |

Logical view size and physical render size are different when device scale is
not `1.0`. Use the getter matching the space you need instead of assuming they
are equal.

!!! note "Changing a property is not always an action"

    Set **HTML Asset** or **HTML File Path** before `BeginPlay`; changing either
    property later does not navigate automatically, so use the matching load
    node. Set **Transparent** before initialization because it is part of view
    creation. After initialization, use **Resize View**, **Set Render Mode**,
    and **Set Max FPS** rather than writing their backing properties directly.
    Changing a physical render cap affects the next calculated resize or view
    initialization; it does not replace the current texture immediately.

### Performance and debug properties

| Blueprint property | Type / default | Meaning |
| --- | --- | --- |
| **Render Mode** | `Magic UI Render Mode`, `Auto` | Requested presentation mode. Read-only to Blueprint property writes; change it with **Set Render Mode**. |
| **Max FPS** | Integer, `60` | Manual output cap from `1`–`240`. Read-only to Blueprint property writes; change it with **Set Max FPS**. |
| **Match Monitor Refresh Rate** | Boolean, `False` | Uses the detected refresh rate instead of **Max FPS** while enabled. The manual value remains the fallback. Monitor detection is currently Windows-specific. |
| **Show UI FPS Counter** | Boolean, `False` | Draws completed UI-texture presentations per second in the top-right of a native **Magic UI** widget. The overlay is not baked into `Get Rendered Texture`. |
| **Max Render Width** | Integer, `7680` | Physical backing-width cap, clamped to `64`–`7680`. |
| **Max Render Height** | Integer, `4320` | Physical backing-height cap, clamped to `64`–`7680`. |

The runtime also enforces a total physical limit of `3840 × 2160` pixels
(8,294,400 pixels). A wide or tall render can therefore be scaled below the
individual width/height caps while preserving its aspect ratio.

### Events

Add these from the component's **Details > Events** section or bind them in a
Blueprint graph.

| Event | Output pins | When it fires |
| --- | --- | --- |
| **On Page Loaded** | `URL` (String) | The document load completed. JavaScript bridge calls can begin here. This does not guarantee that the first texture upload has finished. |
| **On Load Failed** | `URL`, `Error` (Strings) | A local source could not be prepared, a scheme was rejected, a queue rejected the load, or navigation failed later. |
| **On Page Message** | `Json Value` (String) | JavaScript called `globalThis.magic.postMessage(value)`. The pin contains the value serialized as JSON text. |
| **On JavaScript Result** | `Request Id` (Int64), `Success` (Boolean), `Json Result` (String), `Exception` (String) | Completes an **Evaluate JavaScript Async** request. Match the request ID returned by the node. |
| **On Active Render Mode Changed** | `Active Mode` | The actual CPU or GPU-accelerated presentation leaf changed. Requested and active mode can differ after fallback. |
| **On Runtime Process Crashed** | `Process Kind`, `Details` | Reports a Web Content, Network, Render, or unknown runtime-process failure. It does not automatically recover the process. |

All component delegates are delivered on Unreal's game thread.

### Lifecycle and loading functions

| Blueprint node | Inputs / output | Behavior |
| --- | --- | --- |
| **Initialize UI** | Returns Boolean exec branches | Creates the view if necessary. It does not load content by itself. Repeated calls return success for the existing view. |
| **Shutdown UI** | — | Destroys this component's view and clears its current texture, page, focus, and runtime state. This does not make the process-wide runtime safely reloadable. |
| **Is Initialized** | Returns Boolean | `True` after a runtime view ID was accepted. It says nothing about document or frame readiness. |
| **Is Page Ready** | Returns Boolean | `True` after the current page load completed. It becomes false while a new load is pending. |
| **Load HTML From Asset** | `Web Asset`; returns Boolean exec branches | Prepares and queues an imported HTML MagicUI Web File. It initializes the view when necessary. |
| **Load HTML From File** | `File Path`; returns Boolean exec branches | Creates an immutable snapshot of a supported source folder in Editor/PIE. Disabled outside Editor builds. |
| **Load HTML From String** | `HTML Content`; returns Boolean exec branches | Queues inline HTML with MagicUI's built-in local base URL. Best for self-contained content; it does not add sibling files to the inline bundle. |
| **Load URL** | `URL`; returns Boolean exec branches | Accepts only an internal, case-sensitive `magic://bundle/...` URL. A value without `://` is treated as **Load HTML From File**. Remote, `file:`, `data:`, and arbitrary schemes are rejected. |
| **Reload** | — | Repeats the component's current asset, file, string, or internal-bundle load. In an active immutable Editor session, it cannot admit a newly changed loose-file digest; restart PIE instead. |
| **Stop Loading** | — | Requests that the active navigation stop. The node has no completion output. |

!!! tip "Normal beginner path"

    Assign the imported HTML asset in Details and let `BeginPlay` load it. Use
    explicit load nodes only when the game genuinely switches pages at runtime.

### Toggle Magic UI

**Toggle Magic UI** changes an already-created `User Widget` between shown and
hidden states and applies the matching local-player input/cursor policy. It
does not create the widget.

| Pin | Default | Purpose |
| --- | --- | --- |
| **Widget Reference** | — | Existing widget instance to show or hide. |
| **Player Controller** | — | Viewport-backed local controller in the same world. |
| **Is Now Shown** | Output | Resulting visible state. Check **Success** separately. |
| **Shown Visibility** | `Visible` | Must be `Visible`, `Hit Test Invisible`, or `Self Hit Test Invisible`. |
| **Hidden Visibility** | `Collapsed` | Must be `Hidden` or `Collapsed`. |
| **Input Mode When Shown** | `Game and UI` | Input mode applied on show. |
| **Input Mode When Hidden** | `Game Only` | Input mode applied on hide. |
| **Show Mouse Cursor When Shown** | `True` | New controller cursor state on show. |
| **Show Mouse Cursor When Hidden** | `False` | New controller cursor state on hide. |
| **Add To Viewport If Needed** | `True` | Adds the widget the first time it is shown if it is not already in the viewport. |
| **Z Order** | `0` | Viewport Z order used by that first add. Advanced pin. |
| **Mouse Lock Mode** | `Do Not Lock` | Passed to Game-and-UI or UI-only input mode. Advanced pin. |
| **Hide Cursor During Capture** | `False` | Passed to Game-and-UI input mode. Advanced pin. |
| **Flush Input** | `False` | Requests an input flush while changing modes. Advanced pin. |
| **Success** | Output | `False` for invalid references, mismatched worlds/owners, a non-local controller, invalid visibility/mode values, or a failed viewport add. |

Hiding the widget also clears MagicUI view focus. The component, widget, and
controller must share one world; a widget already owned by a different player
is rejected. See [Show and hide a menu](../guides/show-hide-menu.md).

### View and texture functions

| Blueprint node | Returns / inputs | Meaning |
| --- | --- | --- |
| **Resize View** | `Width`, `Height` | Queues a live logical resize without recreating the document. Dimensions are constrained to safe limits. |
| **Get View Size** | Int Point | Current requested/constrained logical CSS size. |
| **Get Render Size** | Int Point | Physical pixel size of the last displayed texture upload. |
| **Get View Device Scale** | Float | Device scale currently requested for the live view. |
| **Get Displayed View Size** | Int Point | Logical viewport corresponding to the texture currently exposed to Blueprint. Use this for manual input mapping. |
| **Get Displayed View Device Scale** | Float | Device scale corresponding to the currently exposed texture. |
| **Get Rendered Texture** | Texture 2D | Current straight-alpha sRGB texture, or `None` before the first completed upload. |
| **Has Displayed Frame** | Boolean | `True` only after at least one render-thread upload completed and a public texture exists. |

There is no Blueprint **On Frame Ready** delegate. For a custom material, poll
**Has Displayed Frame**, fetch the texture, and check again after resize because
a completed size change can replace the texture object. The native UMG widget
handles this itself. Follow [Render texture and world-space UI](../guides/render-texture.md).

### Render and frame-rate functions

| Blueprint node | Inputs / return | Meaning |
| --- | --- | --- |
| **Set Render Mode** | `New Render Mode`; Boolean exec branches | Stores the request before initialization or queues a live mode change. GPU can fall back to CPU when the active RHI is unsuitable. |
| **Get Requested Render Mode** | Render Mode | Returns Auto, CPU, or GPU (Accelerated) as requested. |
| **Get Active Render Mode** | Active Render Mode | Returns None before activation, then the actual CPU or GPU presentation leaf. |
| **Is GPU Accelerated** | Boolean | Convenience check for active mode being GPU (Accelerated). This means accelerated presentation, not GPU browser rasterization. |
| **Set Max FPS** | `New Max FPS`; Boolean exec branches | Accepts only `1`–`240`; stores or queues the cap. |
| **Get Max FPS** | Integer | Manual configured cap, even while monitor matching is authoritative. |
| **Set Match Monitor Refresh Rate** | Boolean | Enables/disables monitor matching. |
| **Is Matching Monitor Refresh Rate** | Boolean | Returns the configured monitor-match state. |
| **Get Detected Monitor Refresh Rate Hz** | Float | Current or last-valid fractional rate; `0` means none detected yet. |
| **Get Effective Max FPS** | Integer | Active cap after monitor detection, rounding, fallback, and `1`–`240` clamping. |
| **Set Show UI FPS Counter** | Boolean | Enables/disables FPS measurement and the native-widget overlay. |
| **Is UI FPS Counter Enabled** | Boolean | Returns the configured counter state. |
| **Get UI Frames Per Second** | Float | Completed public-texture presentations per second, or `0` while disabled/unprimed. |

### Input-state functions

| Blueprint node | Returns | Meaning |
| --- | --- | --- |
| **Has Keyboard Input Focus** | Boolean | Whether an editable HTML control currently has focus according to the latest runtime state. |
| **Is Mouse Over Interactive Element** | Boolean | Latest asynchronous interactive-cursor state used by transparent automatic input routing. |
| **Is Rendered Frame Opaque** | Boolean | Whether the latest displayed frame was reported fully opaque. |

### JavaScript and manual-input functions

| Blueprint node | Inputs / output | Behavior |
| --- | --- | --- |
| **Evaluate JavaScript Async** | `Script`; returns Int64 request ID | Queues source for the main page and completes through **On JavaScript Result**. `0` means immediate rejection. |
| **Post Message To Page** | `Json Value`; Boolean exec branches | Queues a non-empty, valid JSON value string. JavaScript receives the parsed value in a `magic-message` DOM event. |
| **Handle Mouse Move** | `Position` (Vector 2D) | Sends logical view coordinates. Negative coordinates are useful for a pointer-leave condition. |
| **Handle Mouse Button** | `Position`, `Button`, `Pressed`; Boolean exec branches | Queues a supported button press/release at logical coordinates. `None` is rejected. |
| **Handle Mouse Wheel** | `Delta X`, `Delta Y` | Sends pixel-mode wheel deltas at the runtime's most recent pointer position. Send a mouse move first. |
| **Handle Key Event** | `Key`, `Pressed`, `Is Repeat=False` | Sends a DOM-style key down/up mapped from an Unreal `Key`. |
| **Handle Text Input** | `Text` | Inserts text separately from key commands. Empty text is ignored. |
| **Set View Focus** | `Focused` | Gives or removes view focus for a custom input route. |

The **Magic UI** UMG widget calls these input nodes internally. Call them
yourself only for a render texture or other custom presentation. Manual input
must use **Get Displayed View Size**, not physical **Get Render Size**, and must
manage game-input consumption, pointer capture, focus, key releases, and text
input. Ordinary Unicode insertion works; complete OS IME composition is not yet
implemented.

For the raw two-way bridge, see [JavaScript bridge](../guides/javascript-bridge.md).

## Magic UI widget

`Magic UI` (`UMagicUIWidget`) appears in the UMG Designer palette. It paints a
component's texture and performs the normal Slate-to-browser input mapping.

### Properties

| Blueprint property | Type / default | Meaning |
| --- | --- | --- |
| **Source Component** | Magic UI Component, empty | Component whose page, texture, and input endpoint this widget uses. It is exposed on spawn. |
| **Resize View To Widget** | Boolean, `False` | Coalesces stable widget-size changes into live view resizes. Ignored while the component has **Auto Resize To Screen** enabled. |
| **Preserve Aspect Ratio** | Boolean, `True` | Aspect-fits a fixed view instead of stretching it. Automatic screen sizing always covers exactly. |
| **Mouse Input Routing** | Input Routing Mode, `Automatic` | Controls pointer forwarding and whether the widget consumes the event. |
| **Keyboard Input Routing** | Input Routing Mode, `Automatic` | Controls key/text forwarding and whether the widget consumes the event. |

### Function

| Blueprint node | Input | Meaning |
| --- | --- | --- |
| **Set Source Component** | Magic UI Component | Changes the source at runtime and resynchronizes the native widget. |

### Input-routing modes

| Mode | Mouse behavior | Keyboard behavior |
| --- | --- | --- |
| **Automatic (Recommended)** | A fully opaque ready page is modal. A transparent page consumes only where the latest cursor state says HTML is interactive. Blank/loading/failed content passes through. Letterbox space passes through. | Consumes while an editable HTML control has focus (with a short optimistic handoff after interaction). |
| **Always** | Consumes input over the full widget, including letterbox and blank/loading states. Best for deliberately modal menus. | Always consumes and forwards when the view can receive input. |
| **Never** | Still forwards movement so CSS hover/cursor state can update, but does not forward actionable buttons/wheel or block the game. | Does not forward or consume key/text input. |

Input is forwarded only after the component is initialized, the page is ready,
a displayed frame exists, and the texture is valid. On transparent pages,
automatic click routing uses cursor information returned asynchronously by the
runtime. A pointer that enters and clicks in the same tick can therefore let
the first click reach the game before the interactive cursor state arrives.

Once the widget forwards a mouse press, it also forwards the matching release
so the page does not retain a stuck button. Use `pointer-events: none` on purely
decorative transparent HTML and an interactive CSS cursor such as `pointer` or
`text` on actual controls. See [Magic UI widget and input](../guides/widget-and-input.md)
and [Transparency and pass-through](../guides/transparency.md).

## MagicUI Web File

`MagicUI Web File` (`UMagicUIWebAsset`) is the imported Content Browser asset.
Its Blueprint surface is intentionally read-only; author the linked file in an
external editor.

| Blueprint property/function | Type | Meaning |
| --- | --- | --- |
| **Web Asset Type** | Web Asset Type enum | HTML, CSS, or JavaScript as determined by the imported extension. |
| **Source File Name** | String | Original filename including extension, such as `index.html`. |
| **Source Text** | String | Latest cached UTF-8 main-file text used by cooked builds. The custom asset Details panel hides this large field. |
| **Bundled File Count** | Integer | Number of files currently cached for cooking. An HTML import includes supported files recursively beneath its source folder. |
| **Is HTML** | Boolean | `True` only for an `.html` or `.htm` asset suitable as a component entry page. |

Editor-only import data and the binary bundled-file array are not Blueprint
properties. The asset's Details panel instead provides **Open in External
Editor**, **Reveal in File Explorer**, and **Refresh Packaged Copy**. Read the
snapshot and cook rules in [Supported content](supported-content.md).

## MagicUI Event component

`MagicUI Event` (`UMagicUIEventComponent`) is a Blueprint-spawnable Actor
Component that adds named JSON events over a selected Magic UI Component. It
does not own another browser view.

### Properties and events

| Blueprint property/event | Type / default | Meaning |
| --- | --- | --- |
| **Source Magic UI Component** | Component reference, empty | Details-panel picker for a Magic UI Component on the same actor. Blueprint-readable. |
| **Resolved Source Magic UI Component** | Component reference, transient | Runtime source currently bound by the component. |
| **Auto Find Source Component** | Boolean, `True` | If the source is unset, resolves it only when the owner has exactly one Magic UI Component. Multiple candidates are an error. |
| **Max Event Bytes** | Integer, `65536` | Complete protocol-envelope limit, clamped to `1024`–`65536` bytes. |
| **Listen For All Events** | Boolean, `True` | Receives every valid simple named event when enabled. |
| **Event Names To Listen For** | String array | Exact, case-sensitive incoming allowlist used when listen-all is off. |
| **On Event Received** | `Event Name`, `Json Payload` | Fires on the game thread after a valid, allowed event arrives from JavaScript. |
| **On Event Error** | `Event Name`, `Error` | Fires on the game thread for immediate validation/readiness errors and later asynchronous preparation/queue errors. |

### Functions

| Blueprint node | Inputs / output | Meaning |
| --- | --- | --- |
| **Set Source Magic UI Component** | Component | Changes and rebinds the exact source. This supports a component owned by another actor at runtime. |
| **Get Source Magic UI Component** | Returns component | Returns the resolved source, not the editable component-reference structure. |
| **Emit Event** | `Event Name`, `Json Payload`; Boolean exec branches | Queues a named event to JavaScript. An empty payload is JSON `null`; otherwise the text must be one complete strict JSON value. `True` means accepted for asynchronous preparation. |
| **Add Event Listener** | `Event Name`; returns Boolean | Adds one exact name and disables listen-all. Re-adding an existing valid name is idempotent. |
| **Remove Event Listener** | `Event Name` | Removes one allowlist entry. It does not block the name while listen-all remains enabled. |
| **Clear Event Listeners** | — | Disables listen-all and empties the allowlist, so no simple events are received. |
| **Set Listen For All Events** | Boolean | Changes catch-all behavior. |
| **Is Listening For Event** | `Event Name`; returns Boolean | Evaluates the effective catch-all/allowlist filter. |

Event names must be non-empty, no longer than 256 characters, and contain no
control characters. Each component allows at most 64 active-plus-queued items
per direction and runs at most one background preparation task per direction.
Ordering is preserved within that component and direction. Ordering is not
defined across separate event components or raw bridge calls. Delivery is
at-most-once: there is no acknowledgement, retry, or replay.

Wait for **On Page Loaded** before emitting. A `True` result means the event was
accepted for preparation, not that JavaScript handled it. Follow the complete
[MagicUI Event component guide](../guides/magicui-events.md).

## MagicUI Gameplay Tag Event component

`MagicUI Gameplay Tag Event` (`UMagicUIGameplayTagEventComponent`) derives from
MagicUI Event and uses registered `Gameplay Tag` values for event names.

It inherits these visible common properties and events:

- **Source Magic UI Component** and **Resolved Source Magic UI Component**;
- **Auto Find Source Component** and **Max Event Bytes**;
- **On Event Error**; and
- **On Event Received**, which fires with the tag's canonical dotted name
  before the typed event is broadcast.

The simple listen-all/name-array category and simple event functions are hidden
for this component. Generic emission is rejected by its typed implementation;
use the Gameplay Tag nodes below.

### Typed properties, event, and functions

| Blueprint property/function | Type / output | Meaning |
| --- | --- | --- |
| **Gameplay Tags To Listen For** | Gameplay Tag Container | Exact registered incoming allowlist. Empty means receive none; selecting a parent does not include children. |
| **On Gameplay Tag Event Received** | `Event Tag`, `Json Payload` | Fires on the game thread with the registered typed tag after the inherited simple event delegate. |
| **Emit Gameplay Tag Event** | `Event Tag`, `Json Payload`; Boolean exec branches | Queues the tag's dotted name to JavaScript. The tag must be valid; payload and asynchronous return rules match **Emit Event**. |
| **Add Gameplay Tag Listener** | Gameplay Tag | Adds an exact incoming tag. |
| **Remove Gameplay Tag Listener** | Gameplay Tag | Removes an exact incoming tag. |
| **Clear Gameplay Tag Listeners** | — | Empties the incoming tag container, so none are received. |
| **Is Listening For Gameplay Tag** | Gameplay Tag; returns Boolean | Tests exact membership in the incoming container. |

JavaScript supplies a dotted string, but it cannot register or create Gameplay
Tags. Incoming names match selected registered tags case-insensitively, and the
typed delegate returns Unreal's canonical tag. See
[Gameplay Tag events](../guides/gameplay-tag-events.md).

## Blueprint enums

### Magic UI Render Mode

| Value | Meaning |
| --- | --- |
| **Auto (Recommended)** | Uses GPU-accelerated presentation when the active RHI meets the capability gate, otherwise CPU. |
| **CPU** | Converts/presents the shared CPU-rasterized frame on the CPU path. |
| **GPU (Accelerated)** | Requests Unreal-RHI alpha conversion/composition. It is not GPU browser rendering and can fall back to CPU. |

### Magic UI Active Render Mode

`None`, `CPU`, or `GPU (Accelerated)`. This reports the actual presentation
leaf, not the request.

### Magic UI Input Routing Mode

`Automatic (Recommended)`, `Always`, or `Never`. See the widget routing table
above.

### Magic UI Mouse Button

`None`, `Left`, `Middle`, `Right`, `Back`, and `Forward`. `None` cannot be sent
as an actionable button.

### Magic UI Player Input Mode

`Do Not Change`, `Game Only`, `Game and UI`, and `UI Only`. These values are
used by **Toggle Magic UI**.

### Magic UI Runtime Process Kind

`Unknown`, `Web Content`, `Network`, and `Render`. This enum categorizes crash
diagnostics; the `Network` name does not enable remote navigation.

### Magic UI Web Asset Type

`HTML`, `CSS`, and `JavaScript`. `Unknown` exists internally but is hidden from
normal Blueprint enum selection.

## What is intentionally not a Blueprint API

The process-wide runtime manager is an internal owner; normal Blueprints do not
receive a manager reference. Diagnose acceleration through **Get Active Render
Mode** and **Is GPU Accelerated** on the component. Internal frame sequence and
FPS paint-revision counters are also C++ implementation details, not Blueprint
nodes.
