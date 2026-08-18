# Troubleshooting

Start with the smallest observable chain:

```text
plugin loaded
  → component initialized
  → local asset prepared
  → On Page Loaded
  → Has Displayed Frame
  → widget has Source Component
  → input routing/focus
  → JavaScript events
```

Bind **On Load Failed** before experimenting. Its Error pin is usually more
useful than adding an arbitrary delay.

## Fast symptom table

| Symptom | Likely cause | Fix |
|---|---|---|
| MagicUI does not appear in Plugins | Folder nesting is wrong or `MagicUI.uplugin` was not copied. | Verify `<Project>/Plugins/MagicUI/MagicUI.uplugin`, then restart. |
| “Missing or incompatible modules” | Plugin package does not match UE/platform or is incomplete. | Use the UE 5.8 host-platform package and extract it completely. |
| Canvas is blank | UMG child has no runtime Source Component. | Pass the Actor component into Create Widget and call Set Source Component in Event Construct. |
| On Load Failed says file is missing | Imported asset's linked source moved, or loose path is wrong. | Restore/reimport the link; prefer an imported HTML asset. |
| On Page Loaded fires but texture is null | First render upload has not completed. | Wait until Has Displayed Frame is true before sampling. |
| Editor shows old HTML | Current PIE/view session uses an immutable snapshot. | Stop PIE/destroy views, save source, then start a new session. |
| Packaged game shows no page | Component depends on HTML File Path or uncooked content. | Import HTML, assign HTML Asset, refresh packaged copy, save, and recook. |
| Remote URL fails | Remote navigation is intentionally disabled. | Package content locally as an imported MagicUI Web File. |
| Button does not click | Player input mode/cursor/routing or CSS pointer intent is wrong. | Use Game and UI + cursor shown; test Mouse Routing Always; check `pointer-events` and cursor CSS. |
| Text input does not type | Widget lacks focus, Keyboard routing is Never, or player is Game Only. | Use Game and UI/UI Only, Automatic or Always keyboard routing, and click/focus the input. |
| Text is soft | Physical render is smaller than displayed region or scaling owners conflict. | Use Auto Resize To Screen + Auto DPI for full screen; inspect Get Render Size. |
| GPU requested, active mode is CPU | Active RHI cannot use accelerated presenter or it fell back. | Use Auto/CPU or test a supported RHI; inspect On Active Render Mode Changed. |
| UI FPS is below Max FPS | Page is static, Unreal/display is capped, or host work cannot sustain it. | Static zero is correct; otherwise inspect effective cap, game FPS, DPI/size, and damage logs. |
| `Emit Event` returns false | Invalid name/source, page not ready, or component queue is full. | Wait for Page Loaded, validate source/name, and bind On Event Error. |
| Event emit returns true but JS sees nothing | True means async preparation accepted, not delivered; listener may not exist. | Load helper before app JS, register listener early, and watch On Event Error. |
| Gameplay Tag event is ignored | Tag is unregistered or absent from receive container. | Register the tag and select it under Gameplay Tags To Listen For. |
| Material never updates after resize | Texture object was replaced and custom presenter kept old reference. | Re-read Get Rendered Texture after Has Displayed Frame. |

## Blank screen diagnosis

### 1. Verify content

- The component's **HTML Asset** is the imported HTML MagicUI Web File.
- Double-click the asset: **Status** is **Exists**.
- **On Load Failed** is bound and does not report a bundle/path error.
- The page uses supported local relative resources.

### 2. Verify the runtime connection

- The correct Actor is placed or spawned.
- Its Magic UI Component reaches **Is Initialized**.
- `Create Widget` receives that same component on your exposed source pin.
- Event Construct calls **Set Source Component** on the actual Magic UI child.
- The returned User Widget is added to the viewport.

### 3. Verify layout

- The Magic UI child is visible and not zero-size.
- In a Canvas Panel it has full-stretch anchors and zero offsets for full screen.
- A parent animation or visibility setting is not hiding/collapsing it.
- Component and widget are not fighting over resize ownership.

### 4. Verify milestones

Print these states at a slow diagnostic interval or in response to an input key:

```text
Is Initialized
Is Page Ready
Has Displayed Frame
Get View Size
Get Render Size
Get Active Render Mode
```

Avoid printing every Tick in a normal session.

## Page and resource failures

### Source changes do not appear

The source snapshot is frozen while a session has live views. Stop PIE, make
the edit, save, and start PIE again. **Reload** alone reloads the frozen current
bundle.

### CSS/JS/image is missing

Check:

- the file sits beneath the imported entry HTML directory;
- its extension is supported;
- each file is at most 1 MiB;
- path case matches exactly;
- no case-only duplicate exists;
- the path uses forward slashes and no traversal; and
- the asset's packaged copy was refreshed before cooking.

### Load URL reports policy denied

Only internal, prepared, case-sensitive `magic://bundle/...` URLs are allowed.
There is no Project Settings allowlist for remote origins.

## Input diagnosis

For a modal test, temporarily set:

```text
Mouse Input Routing = Always
Keyboard Input Routing = Always
Player input mode = Game and UI
Show Mouse Cursor = true
```

If that works, return to Automatic and fix transparent-region CSS/focus policy.
For a transparent overlay, confirm interactive elements use
`pointer-events:auto` and `cursor:pointer`/`cursor:text`, while decorations use
`pointer-events:none`.

Remember the known limitation: cached transparent hit state is asynchronous,
so the exact first click immediately after pointer entry can race.

## Messaging diagnosis

### Raw bridge

- JS sends the value itself with `globalThis.magic.postMessage(value)`.
- Unreal receives compact JSON text through **On Page Message**.
- Unreal sends a complete valid JSON value string through **Post Message To
  Page**.
- JS receives an already parsed value in `event.detail`.
- send from Unreal only after **On Page Loaded**.

### Named event bridge

- `magicui-events.js` loads before the application script.
- the event component points at the correct view.
- simple event names match case exactly.
- **Listen For All Events** is on, or the exact name is in the allowlist.
- **On Event Error** is bound.
- Json Payload is a complete JSON value; a string requires quotes.
- the complete UTF-8 envelope fits **Max Event Bytes**.

See [MagicUI Event component](../guides/magicui-events.md) for an end-to-end
test.

## Rendering and performance diagnosis

Compare **Get Requested Render Mode** with **Get Active Render Mode**. CPU
active under Auto is a valid fallback. GPU (Accelerated) is not a GPU browser
renderer; it accelerates only the Unreal presentation conversion.

Turn on bounded damage logging temporarily:

```text
MagicUI.CPU.LogDamage 1
```

Search the Output Log for `MAGICUI_DAMAGE_PRESENTED`. Each line includes the
physical size, changed rectangle, patch bytes, full-frame equivalent, opacity,
and GPU conversion flag. Turn it back off:

```text
MagicUI.CPU.LogDamage 0
```

The widget's optional UI FPS counter measures completed MagicUI presentations.
A static page returning to 0 FPS is expected and efficient.

## Logs and failure events

Open:

<div class="ui-path"><span>Window</span><i></i><span>Developer Tools</span><i></i><span>Output Log</span></div>

Search for `LogMagicUI`, `On Load Failed` Error text, event errors, and runtime
crash details. Useful delegates are:

- **On Load Failed**
- **On Event Error**
- **On JavaScript Result** with Success=false and Exception
- **On Active Render Mode Changed**
- **On Runtime Process Crashed**

## Restart after native plugin changes

MagicUI's native runtime and owner thread are process-lifetime, and dynamic
module reload is unsupported. After replacing/updating the plugin package,
close Unreal Editor completely, verify the files, and start it again. A hot
reload is not a valid test of replaced native libraries.

If a current package still fails before any component initializes, record the
exact Unreal version, platform, plugin package identity, complete LogMagicUI
error, and the smallest reproduction graph before requesting support.
