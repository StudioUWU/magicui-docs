# Transparency and pass-through

A transparent MagicUI HUD can draw buttons and status elements over gameplay
without turning the entire full-screen Canvas into an invisible input shield.
This requires both component configuration and intentional CSS.

## Enable page alpha

On the **Magic UI Component**, enable **Transparent**. Then remove opaque page
backgrounds in CSS:

```css
html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  background: transparent;
}
```

If the component permits transparency but `body` paints an opaque background,
the rendered frame is still opaque. Use **Is Rendered Frame Opaque** when
diagnosing the result.

## Mark interactive and decorative regions

For Automatic mouse routing, make the intent visible to both CSS and MagicUI's
cached cursor state:

```css
/* Full-screen visual layer does not take gameplay clicks. */
.hud-layer,
.decoration {
  pointer-events: none;
}

/* Individual controls opt back into interaction. */
button,
a,
input,
textarea,
.interactive {
  pointer-events: auto;
}

button,
a,
[role="button"] {
  cursor: pointer;
}

input,
textarea {
  cursor: text;
}
```

`pointer-events: none` can be applied to a decorative parent, then interactive
children can opt back in with `pointer-events: auto`.

## Use Automatic input routing

On the UMG **Magic UI** widget:

```text
Mouse Input Routing = Automatic (Recommended)
Keyboard Input Routing = Automatic (Recommended)
```

Automatic routing behaves differently based on current page state:

- an opaque page is modal over the widget;
- a transparent page consumes pointer actions over cached interactive HTML;
- transparent/non-interactive regions continue to gameplay;
- keyboard is consumed while editable HTML has focus;
- aspect-fit letterbox bars pass through; and
- a blank/loading view does not become an invisible gameplay shield.

Use **Always** for a modal pause/main menu where all clicks must belong to UI.
Use **Never** for a decorative page that needs CSS hover movement but must not
receive actionable clicks, wheel, or keys.

## Full-screen transparent HUD example

```html
<body>
  <div class="hud-layer">
    <section class="status decoration">Health <span id="health">100</span></section>
    <button class="inventory interactive" id="inventory">Inventory</button>
  </div>
</body>
```

```css
.hud-layer {
  position: fixed;
  inset: 0;
  pointer-events: none;
}

.status {
  position: absolute;
  top: 24px;
  left: 24px;
}

.inventory {
  position: absolute;
  right: 24px;
  bottom: 24px;
  pointer-events: auto;
  cursor: pointer;
}
```

Gameplay input passes through the empty center, while the Inventory button is
clickable.

## Transparency and materials

The public rendered texture always uses normal **straight alpha in sRGB**,
whether the active presenter is CPU or GPU (Accelerated). Do not enable a
premultiplied-alpha material workaround. Use a normal translucent or UI
material appropriate to the Unreal surface receiving it.

GPU (Accelerated) uses a private premultiplied input internally, converts it,
and exposes the same public texture contract as CPU mode.

## First-click timing caveat

Transparent Automatic routing is intentionally nonblocking, so it relies on
cursor state received asynchronously from the page. On the exact first tick
that the pointer enters a control, a click can race the new interactive state.
Later clicks use the updated cache.

For a screen where the first click must always be consumed, prefer **Always**.
For a gameplay HUD, keep Automatic and avoid controls that appear directly
under an already-pressed pointer.

## Troubleshooting transparency

| Symptom | Check |
|---|---|
| Black/colored rectangle behind the page | Component Transparent and `html, body` backgrounds. |
| Whole HUD blocks gameplay | Mouse routing is Automatic; overlay containers use `pointer-events:none`. |
| Button hover works but click passes through | Give the control `pointer-events:auto` and an interactive cursor, then move over it before testing. |
| Text field never receives typing | Player input mode allows UI focus; Keyboard routing is not Never; the field has focus. |
| Letterbox bar blocks input | Preserve Aspect Ratio and Automatic routing; Always intentionally consumes it. |
| Material edge colors look dark | Remove premultiplied-alpha compensation; the public texture is straight alpha. |

See [Magic UI widget and input](widget-and-input.md) for all routing modes.
