# 2. Create the first HTML page

Create a dedicated source folder outside Unreal's Content Browser. Keeping each
web UI in its own folder is important because importing an HTML entry file
collects supported files recursively from the entry file's whole directory.

## Create the folder

A simple project-owned layout is:

```text
<YourProject>/WebSource/
└── GettingStarted/
    └── index.html
```

`WebSource` is ordinary source content, not a special Unreal folder. Add it to source
control with the rest of the project.

## Add the page

Create `index.html`, paste the complete example below, and save it as UTF-8.

```html title="WebSource/GettingStarted/index.html" linenums="1"
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My first MagicUI screen</title>
  <style>
    * { box-sizing: border-box; }
    html, body { width: 100%; height: 100%; margin: 0; }
    body {
      display: grid;
      place-items: center;
      overflow: hidden;
      color: #f7f4ff;
      background:
        radial-gradient(circle at 70% 20%, #8b5cf655, transparent 35%),
        linear-gradient(145deg, #101525, #1c1434);
      font: 18px/1.5 system-ui, sans-serif;
    }
    main {
      width: min(560px, 86vw);
      padding: 40px;
      border: 1px solid #ffffff22;
      border-radius: 24px;
      background: #111827cc;
      box-shadow: 0 24px 70px #0008;
      text-align: center;
    }
    .eyebrow { color: #c4b5fd; letter-spacing: .16em; }
    h1 { margin: 8px 0 12px; font-size: clamp(32px, 6vw, 64px); }
    #status { min-height: 1.5em; color: #a7f3d0; }
    button {
      margin-top: 12px;
      padding: 13px 22px;
      border: 0;
      border-radius: 999px;
      color: white;
      background: #7c3aed;
      font: inherit;
      font-weight: 700;
      cursor: pointer;
    }
    button:hover { background: #8b5cf6; transform: translateY(-1px); }
  </style>
</head>
<body>
  <main>
    <div class="eyebrow">MAGICUI · UNREAL ENGINE</div>
    <h1>It renders!</h1>
    <p>This page came from an imported HTML asset.</p>
    <button id="hello" type="button">Send hello to Unreal</button>
    <p id="status">Waiting for a click…</p>
  </main>

  <script>
    const button = document.querySelector("#hello");
    const status = document.querySelector("#status");

    button.addEventListener("click", () => {
      if (!globalThis.magic || typeof globalThis.magic.postMessage !== "function") {
        status.textContent = "Preview mode: the Unreal bridge is not available.";
        return;
      }

      globalThis.magic.postMessage({
        type: "tutorial.hello",
        message: "Hello from JavaScript"
      });
      status.textContent = "Message queued for Unreal.";
    });

    window.addEventListener("magic-message", (event) => {
      status.textContent = `Unreal replied: ${JSON.stringify(event.detail)}`;
    });
  </script>
</body>
</html>
```

[Download this `index.html`](../assets/examples/starter-ui/index.html){ .md-button download="index.html" }

## Preview it in a desktop browser

Double-click the file or use your editor's preview server. You should see a
dark purple card with **It renders!** and one button. Clicking outside Unreal
will report that the bridge is unavailable; that is expected. The bridge is
installed by MagicUI when the page runs inside the plugin.

<div class="step-result">The page renders correctly in an ordinary browser, and the button changes the status text without a JavaScript error.</div>

## What the bridge code means

`globalThis.magic.postMessage(...)` sends a JSON-compatible JavaScript value to
the component's **On Page Message** Blueprint event. Pass the object itself—do
not call `JSON.stringify` unless you intentionally want Unreal to receive a JSON
string value rather than an object.

The `magic-message` listener handles values traveling in the opposite direction.
The event's `detail` field is already the parsed JavaScript value.

You will test both directions after the screen is visible. The complete rules
are in [JavaScript bridge](../guides/javascript-bridge.md).

## When you split the page into files

Relative paths work as expected when the files remain beneath this dedicated
folder:

```text
GettingStarted/
├── index.html
├── css/main.css
├── js/app.js
├── images/icon.png
└── fonts/inter.woff2
```

Use references such as `css/main.css` and `images/icon.png`. Do not use an
absolute Windows path, `file:///`, or an internet URL; the plugin serves an
immutable local bundle and rejects arbitrary schemes.

Next, [import the HTML file into Unreal](import-html-asset.md).
