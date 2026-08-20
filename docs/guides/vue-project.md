# Build a Vue interface

Use Vue and Vite to author a MagicUI screen, then build it as one
self-contained HTML file. The finished file can be imported directly as a
**MagicUI Web File** and does not need a web server at runtime.

This example has two small routes, uses Bootstrap for basic layout, and sends a
message to Unreal when you select a button.

## Before you begin

Install Node.js `22.12` or newer. This includes the `npm` command used below.
You also need a Magic UI Component ready to receive an imported HTML asset. If
you have not created one yet, complete
[Render your first screen](../getting-started/first-screen.md) first.

## Create the project files

Create a folder for the Vue source outside Unreal's `Content` directory. For
example:

```text
<Project>/WebSource/MagicUIVue/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── ui.html
└── src/
    ├── App.vue
    ├── env.d.ts
    ├── main.ts
    ├── pages/
    │   ├── GameHud.vue
    │   └── MainMenu.vue
    └── router/
        └── index.ts
```

The filenames in this guide are important. In particular, `ui.html` is the
Vite entry page and becomes `dist/ui.html` after the build.

### Add `package.json`

Create `package.json` in `MagicUIVue`:

```json
{
  "name": "magicui-vue-ui",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host 127.0.0.1 --open /ui.html",
    "type-check": "vue-tsc --noEmit",
    "build": "npm run type-check && vite build",
    "preview": "vite preview --host 127.0.0.1"
  },
  "dependencies": {
    "bootstrap": "5.3.8",
    "vue": "^3.5.41",
    "vue-router": "^4.6.4"
  },
  "devDependencies": {
    "@types/node": "^26.2.0",
    "@vitejs/plugin-vue": "^6.0.8",
    "typescript": "5.9.3",
    "vite": "^8.2.1",
    "vite-plugin-singlefile": "^2.3.3",
    "vue-tsc": "^3.3.9"
  }
}
```

The `build` script checks the TypeScript and Vue files before Vite creates the
runtime page. Bootstrap is installed locally instead of loaded from a CDN.
`vite-plugin-singlefile` then places Bootstrap's CSS, the Vue runtime, Vue
Router, and the application code inside one HTML file.

### Add `tsconfig.json`

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "lib": ["ES2017", "DOM", "DOM.Iterable"],
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "jsx": "preserve",
    "types": ["vite/client", "node"]
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue", "vite.config.ts"]
}
```

This configuration type-checks Vue single-file components, browser APIs, and
`vite.config.ts`. The `@/` alias points to `src/`; the same alias is configured
in Vite next.

### Add `vite.config.ts`

Create `vite.config.ts`:

```typescript
import { fileURLToPath, URL } from 'node:url'

import vue from '@vitejs/plugin-vue'
import { defineConfig } from 'vite'
import { viteSingleFile } from 'vite-plugin-singlefile'

export default defineConfig({
  base: './',
  plugins: [
    vue(),
    viteSingleFile({
      removeViteModuleLoader: true,
    }),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  build: {
    target: ['es2017', 'safari16.4'],
    cssTarget: 'safari16.4',
    outDir: 'dist',
    emptyOutDir: true,
    sourcemap: false,
    rollupOptions: {
      input: fileURLToPath(new URL('./ui.html', import.meta.url)),
    },
  },
})
```

These settings do the MagicUI-facing build work:

- `base: './'` keeps generated URLs relative to the HTML page.
- `viteSingleFile()` inlines the application code and styles.
- `rollupOptions.input` makes `ui.html` the entry page.
- `target` and `cssTarget` keep the output compatible with MagicUI's browser
  baseline.
- `outDir` places the finished page in `dist` and `emptyOutDir` removes the
  previous build before creating a new one.

### Add `ui.html`

Create `ui.html` in the project root:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="color-scheme" content="dark" />
    <title>MagicUI Vue example</title>
  </head>
  <body class="bg-dark text-light">
    <div id="app">Loading...</div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Vite uses the absolute `/src/main.ts` path while running its development
server. During a production build, the single-file plugin replaces this with
the compiled application inside `dist/ui.html`.

### Add the TypeScript files

Create `src/env.d.ts` so TypeScript knows about MagicUI's page bridge:

```typescript
/// <reference types="vite/client" />

interface MagicUIBridge {
  postMessage(value: unknown): void
}

declare global {
  var magic: MagicUIBridge | undefined
}

export {}
```

Create `src/main.ts` to load Bootstrap, start the router, and mount the Vue
application:

```typescript
import 'bootstrap/dist/css/bootstrap.min.css'

import { createApp } from 'vue'

import App from '@/App.vue'
import router from '@/router'

async function startApplication(): Promise<void> {
  await router.replace({ name: 'MainMenu' })
  await router.isReady()

  createApp(App).use(router).mount('#app')
}

void startApplication()
```

Bootstrap's CSS is imported from the installed package, so Vite can include it
in the self-contained build. The application selects the main menu before it
mounts, which gives the memory router a known first route.

### Add the router

Create `src/router/index.ts`:

```typescript
import { createMemoryHistory, createRouter } from 'vue-router'

import GameHud from '@/pages/GameHud.vue'
import MainMenu from '@/pages/MainMenu.vue'

export default createRouter({
  history: createMemoryHistory(),
  routes: [
    {
      path: '/',
      name: 'MainMenu',
      component: MainMenu,
    },
    {
      path: '/hud',
      name: 'GameHud',
      component: GameHud,
    },
  ],
})
```

`createMemoryHistory()` keeps the active route in memory. Use it for MagicUI
instead of web history: the page runs from a local bundle rather than an HTTP
server, and its internal route does not need to change the browser URL.

### Add the application shell

Create `src/App.vue`:

```vue
<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="container-fluid min-vh-100 p-3 p-md-4 p-lg-5">
    <header
      class="d-flex flex-column flex-sm-row gap-3 justify-content-between
             align-items-sm-center mb-4"
    >
      <h1 class="h3 mb-0">MagicUI + Vue</h1>

      <nav class="nav nav-pills gap-2" aria-label="Example pages">
        <RouterLink
          class="nav-link"
          active-class="active"
          :to="{ name: 'MainMenu' }"
        >
          Main menu
        </RouterLink>
        <RouterLink
          class="nav-link"
          active-class="active"
          :to="{ name: 'GameHud' }"
        >
          Game HUD
        </RouterLink>
      </nav>
    </header>

    <RouterView />
  </div>
</template>

<style>
html,
body,
#app {
  width: 100%;
  height: 100%;
  margin: 0;
}

body {
  font-family: system-ui, sans-serif;
}
</style>
```

`RouterLink` changes the active Vue route without loading another HTML page.
Bootstrap's `container-fluid`, flex, spacing, and breakpoint classes make the
header stack on narrow views and sit in one row when more width is available.

### Add the routed pages

Create `src/pages/MainMenu.vue`:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const status = ref('Ready')

function notifyUnreal(): void {
  if (globalThis.magic === undefined) {
    status.value = 'Browser preview: the MagicUI bridge is not available'
    return
  }

  try {
    globalThis.magic.postMessage({
      type: 'vue-button-clicked',
      text: 'Hello from Vue',
    })
    status.value = 'Message posted to MagicUI'
  } catch (error) {
    status.value = `Send failed: ${String(error)}`
  }
}
</script>

<template>
  <main class="row g-4 align-items-center">
    <section class="col-12 col-lg-7">
      <p class="text-uppercase text-info fw-semibold mb-2">Main menu</p>
      <h2 class="display-5 fw-bold">A responsive Vue screen</h2>
      <p class="lead text-white-50">
        This content fills the available width and stacks on smaller views.
      </p>
      <button class="btn btn-primary btn-lg" type="button" @click="notifyUnreal">
        Notify Unreal
      </button>
      <p class="alert alert-secondary mt-3 mb-0" role="status">
        {{ status }}
      </p>
    </section>

    <aside class="col-12 col-lg-5">
      <div class="card text-bg-dark border-secondary">
        <div class="card-body p-4">
          <h3 class="h5 card-title">Bootstrap layout</h3>
          <p class="card-text mb-0">
            This card moves beside the menu at the large breakpoint.
          </p>
        </div>
      </div>
    </aside>
  </main>
</template>
```

Create `src/pages/GameHud.vue`:

```vue
<template>
  <main>
    <p class="text-uppercase text-info fw-semibold mb-2">Game HUD</p>
    <h2 class="h4 mb-4">Player status</h2>

    <div class="row g-3">
      <section class="col-12 col-md-8">
        <div class="card text-bg-dark border-secondary h-100">
          <div class="card-body">
            <h3 class="h6 text-white-50">Health</h3>
            <div
              class="progress"
              role="progressbar"
              aria-label="Player health"
              aria-valuenow="82"
              aria-valuemin="0"
              aria-valuemax="100"
            >
              <div class="progress-bar bg-success" style="width: 82%">82%</div>
            </div>
          </div>
        </div>
      </section>

      <section class="col-12 col-md-4">
        <div class="card text-bg-primary h-100">
          <div class="card-body">
            <h3 class="h6">Score</h3>
            <p class="display-6 mb-0">1,250</p>
          </div>
        </div>
      </section>
    </div>
  </main>
</template>
```

The grid uses full-width columns on narrow views and splits them at Bootstrap's
`md` and `lg` breakpoints. No Bootstrap JavaScript is needed for these layout
and component classes.

The browser preview does not provide `globalThis.magic`, so **Notify Unreal**
shows a preview message there. MagicUI installs the bridge before the page
scripts run, and the same button posts an object to the component's **On Page
Message** event inside Unreal.

## Install and preview

Open a terminal in `MagicUIVue`, then install the packages:

```bash
npm install
```

This also creates `package-lock.json`. Keep that file in source control so the
team installs the same resolved package versions.

Start the Vite development server:

```bash
npm run dev
```

Vite opens `/ui.html` in your normal browser. Resize the window to check the
responsive layout, and use **Main menu** and **Game HUD** to check both routes.
Select **Notify Unreal** to confirm that the Vue component is running; the
status explains that the MagicUI bridge is unavailable in the browser preview.

## Build the MagicUI page

Stop the development server, then run:

```bash
npm run build
```

The type check must pass, and the build must create this file:

```text
MagicUIVue/dist/ui.html
```

That one file contains Bootstrap's CSS, the Vue runtime, Vue Router, component
code, and project CSS. Do not import the source `ui.html`; it still points to
`/src/main.ts` and expects Vite's development server.

## Import and test in Unreal

1. Stop PIE.
2. Drag `dist/ui.html` into the Unreal Content Browser.
3. Confirm that Unreal creates a **MagicUI Web File**.
4. Assign that asset to the Magic UI Component's **HTML Asset** property.
5. Add the component's **On Page Message** event and print its **Json Value**.
6. Start PIE and select **Notify Unreal**.

The page changes its status to `Message posted to MagicUI`, and Blueprint
receives a JSON value like this:

```json
{"type":"vue-button-clicked","text":"Hello from Vue"}
```

For more ways to handle the value, see
[Send data between JavaScript and Unreal](javascript-bridge.md).

## Rebuild after an edit

Each time you change the Vue source:

1. run `npm run build`;
2. stop PIE;
3. reimport `dist/ui.html`, or use **Refresh Packaged Copy** on its MagicUI Web
   File asset;
4. save the Unreal asset; and
5. start PIE again.

MagicUI uses an immutable snapshot during a view session, so rebuilding the
file while PIE is running does not update the active page. See
[Web assets and local content](web-assets.md) for the full edit and packaging
workflow.
