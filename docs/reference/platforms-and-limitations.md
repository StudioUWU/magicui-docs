# Platforms and limitations

MagicUI 2.X.X uses a strict local content runtime. This documentation targets
Unreal Engine **5.8**.

## Current platform status

| Target | Supported |
| --- | --- |
| **Windows x64 (Win64)** | Yes |
| **macOS arm64 (Apple silicon)** | Yes; monitor refresh uses the manual **Max FPS** fallback |
| **Linux** | No |
| **Consoles** | No |
| **Mobile** | No |

Obtain a package that matches the exact platform, architecture,
and Unreal version.

## Local content only

The active runtime capability profile is deliberately strict:

| Capability | Status |
| --- | --- |
| Imported/local bundle content | Supported |
| Internal `magic://bundle/...` navigation | Supported |
| Remote HTTP/HTTPS navigation or fetch | Disabled |
| Arbitrary `file:`, `data:`, or custom-scheme navigation | Disabled |
| Persistent browser cache directory | Disabled |
| Persistent browser storage directory | Disabled |

Plan pages to be self-contained, use relative resources, and import the HTML
entry as a MagicUI Web File. See [Supported content](supported-content.md).

## Rendering model

MagicUI always receives a CPU-rasterized WebCore frame. Its render-mode names
select how that shared frame is presented to Unreal:

| Requested mode | Actual behavior |
| --- | --- |
| **Auto (Recommended)** | Selects GPU-accelerated presentation when the active RHI satisfies the capability checks; otherwise uses CPU. |
| **CPU** | Converts the CPU frame to the public Unreal texture on the CPU presentation path. |
| **GPU (Accelerated)** | Uploads the premultiplied CPU patch and uses a small Unreal SM5/RHI pass. |

Use **Get Active Render Mode**, **Is GPU Accelerated**, and **On Active Render
Mode Changed** to observe the result. Do not assume a successful **Set Render
Mode** means that GPU presentation became active. See
[Rendering modes and performance](../guides/rendering-and-performance.md).

## Texture readiness and replacement

View initialization, page load, and texture availability are separate:

```text
Initialize/load accepted
  → On Page Loaded
  → first render-thread upload completes
  → Has Displayed Frame = true
  → Get Rendered Texture is valid
```

**On Page Loaded can fire before the first displayed texture exists.** There is
no Blueprint frame-ready delegate. Custom material users must poll **Has
Displayed Frame** and **Get Rendered Texture**.

At a fixed physical size, damage updates patch one persistent `Texture2D` in
place. A resize, DPI change, reinitialization, or other physical extent change
can create a replacement object; it becomes public only after its complete
upload finishes. A custom material must re-fetch and rebind the texture after
such a change. The native UMG **Magic UI** widget handles paint invalidation and
replacement automatically.

The widget-only FPS counter is drawn by Slate; it is not embedded in the
texture returned to a mesh or custom material. There is no Blueprint GPU
readback API. Follow [Render texture and world-space UI](../guides/render-texture.md)
for the safe binding pattern.

## Size, DPI, and cadence limits

- Logical width and height are constrained to at least 1 and no more than 7,680
  on either axis.
- **Max Render Width** and **Max Render Height** are each clamped to 64–7,680.
- Total physical output is limited to 8,294,400 pixels (`3840 × 2160`), so
  requested dimensions may be proportionally reduced.
- Device scale is clamped to 0.25–8.0.
- **Max FPS** accepts 1–240.
- The component exposes completed public-texture presentation FPS, not page
  JavaScript timers, game FPS, or a promise that every cap will be reached.

Automatic viewport sizing follows the game/PIE viewport and backing scale. For
pixel-exact packaged output, enable Unreal's **Allow High DPI in Game Mode**;
the plugin logs a warning when it is disabled outside the editor.

Monitor refresh detection and window-to-monitor tracking are implemented on
Windows. On other platforms, detection returns no rate and
**Match Monitor Refresh Rate** uses the manual **Max FPS** fallback.

## Input limitations

The native UMG **Magic UI** widget is the supported screen-space input host. It
maps Slate pointer positions, buttons, wheel, key events, text insertion, and
focus into the logical web view.

When you use **Get Rendered Texture** on a mesh or another custom material,
MagicUI cannot infer a ray hit, UV, pointer capture, player, or focus target.
You must translate the hit to logical coordinates and call the component's
manual input nodes yourself. Rendering the texture does not make a world-space
surface interactive. See the [manual input section](../guides/render-texture.md#add-manual-input-when-needed).

Additional caveats:

- Transparent **Automatic** routing depends on asynchronously returned browser
  cursor state. A pointer that enters and clicks in the same tick can let the
  first click pass to gameplay before the interactive state arrives. Use
  **Always** for a modal screen that requires first-click certainty.
- Normal Unicode insertion is supported, but complete Slate/OS IME composition
  is not implemented. Test languages that require composition carefully.
- The widget forwards input only when the view is initialized, the page is
  ready, at least one frame is displayed, and the texture is valid.
- With aspect preservation, Automatic mode lets letterbox regions pass through;
  Always consumes them.
- CSS cursor and `pointer-events` choices are part of transparent hit routing.
  Decorative layers should normally use `pointer-events: none`.

Read [Magic UI widget and input](../guides/widget-and-input.md) and
[Transparency and pass-through](../guides/transparency.md) for the normal UMG
path.

## Bundle and cook limitations

Important content boundaries include:

- one runtime resource is limited to 1 MiB;
- one bundle is limited to 4,096 files and 64 MiB;
- safe ASCII relative paths are required, with no traversal or
  case-insensitive collisions;
- active Editor bundle sessions are immutable—stop/restart PIE after edits;
  and
- the cooked package catalog freezes at process-runtime startup, so assets that
  may be selected later must be staged/referenced before the first view.

The complete extension, path, snapshot, and cooking rules are in
[Supported content](supported-content.md).

## Asynchronous and bounded operations

Runtime commands, events, frames, and host copies use bounded queues. This is
why many Blueprint nodes return an acceptance Boolean or request ID rather than
blocking the game thread.

- A load-node `True` is followed by **On Page Loaded** or **On Load Failed**.
- **Evaluate JavaScript Async** returns `0` on immediate rejection and otherwise
  completes through **On JavaScript Result**.
- **Post Message To Page** and event emission report queue acceptance, not
  JavaScript handling.
- Named events are at-most-once, with no acknowledgement or retry.
- One event component preserves order within each direction, but separate
  components and raw bridge calls have no shared ordering guarantee.

Wait for **On Page Loaded** before sending data. Bind failure/error events while
developing. See the [Blueprint API](blueprint-api.md) and
[MagicUI Event guide](../guides/magicui-events.md).

## Process lifetime and restart requirements

The browser runtime installs process-global WebCore/JavaScriptCore state and
has one accepted owner-thread lifetime. The module pins its native runtime
closure until process exit. Terminal runtime shutdown, module unload, or a
native plugin replacement is not followed by a supported in-process restart.

Close and restart Unreal Editor after updating/unloading the plugin or after a
terminal runtime shutdown. Repeated PIE view creation is a different operation:
PIE can destroy and recreate views while the same process runtime remains
alive. Test repeated view creation and destruction in PIE if your workflow
depends on it.

## Diagnostics

The plugin exposes **On Runtime Process Crashed**, **On Load Failed**, **On
Event Error**, requested/active render-mode queries, UI presentation FPS, and
the `MagicUI.CPU.LogDamage 1` console diagnostic. These features report runtime
state and failures but do not provide automatic crash recovery or persistent
telemetry.

When reporting a problem, include:

1. Unreal version and target configuration;
2. operating system, architecture, GPU, and active RHI;
3. plugin package identity/version;
4. requested and active render modes;
5. load/event error text and the relevant Unreal log section; and
6. whether the issue occurs in Editor, PIE, standalone, or a packaged game.
