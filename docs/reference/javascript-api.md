# JavaScript API reference

This page documents the JavaScript surface exposed by the MagicUI Unreal
plugin and the optional `magicui-events.js` helper shipped with it.

For step-by-step setup, see [Send data between JavaScript and Unreal](../guides/javascript-bridge.md)
and [Use the MagicUI Event component](../guides/magicui-events.md).

## Availability

The bridge is installed on the main page before its authored scripts execute:

```javascript
globalThis.magic
```

Because a browser Window is the global object, `window.magic` refers to the
same value.

```javascript
const isRunningInMagicUI = Boolean(
  globalThis.magic && typeof globalThis.magic.postMessage === "function"
);
```

The bridge does not exist when the HTML file is opened in Chrome, Edge,
Firefox, Safari, or another ordinary browser. There is no
`window.ue.magicui` alias.

## `globalThis.magic`

`magic` is a frozen object installed as a read-only, non-configurable global.
Application code cannot replace it or add methods to it.

Conceptual type declaration:

```typescript
interface MagicUIBridge {
  postMessage(value: JSONValue): void;
}

type JSONPrimitive = string | number | boolean | null;
type JSONValue =
  | JSONPrimitive
  | JSONValue[]
  | { [key: string]: JSONValue };

declare const magic: Readonly<MagicUIBridge>;
```

### `magic.postMessage(value)`

Sends one JSON-compatible JavaScript value from the page to the owning Magic
UI Component.

```javascript
magic.postMessage({
  type: "menu-action",
  action: "start-game",
  sequence: 7
});
```

| Detail | Behavior |
| --- | --- |
| Parameters | Exactly one JSON-compatible value |
| Return value | JavaScript `undefined` after synchronous native admission |
| Unreal receiver | **On Page Message(Json Value)** |
| Unreal value type | Serialized JSON text |
| Delivery | Asynchronous, bounded, at-most-once |

The bridge applies `JSON.stringify` internally. Pass objects and arrays
directly rather than stringifying them first.

Serialization happens before the native call, so `postMessage` throws
synchronously when `JSON.stringify` throws—for example, for a cycle or
`BigInt`. It also throws synchronously if the native bridge rejects admission.
Only JSON-compatible top-level values are supported; values such as
`undefined` or a function do not produce a JSON string and are not valid
messages. On success, the function returns JavaScript `undefined`; there is no
result or acknowledgement channel in that return value.

Normal JavaScript JSON behavior still applies. For example, a finite number
remains a number, unsupported object properties may be omitted, and `NaN` or
infinity serialize as `null`. Prefer deliberately constructed data-only
objects instead of relying on those conversions.

```javascript
try {
  magic.postMessage({ type: "ready", value: 42 });
} catch (error) {
  console.error("The message could not be serialized or admitted", error);
}
```

Do not treat a non-throwing call as an acknowledgement from Blueprint. If
application confirmation matters, send a separate response message carrying
your own request ID.

## `magic-message`

Unreal-to-page messages dispatch a `CustomEvent` on `window`/`globalThis`:

```javascript
function onMagicMessage(event) {
  console.log(event.detail);
}

window.addEventListener("magic-message", onMagicMessage);

// Later, if this code no longer owns the listener:
window.removeEventListener("magic-message", onMagicMessage);
```

`event.detail` is the parsed value represented by the JSON string passed to
Blueprint's **Post Message to Page** node.

| Blueprint JSON input | JavaScript `event.detail` type |
| --- | --- |
| `{"health":100}` | Object |
| `[1,2,3]` | Array |
| `"hello"` | String |
| `42` | Number |
| `false` | Boolean |
| `null` | `null` |

The DOM event does not carry a return channel. Send a separate page message if
the page needs to acknowledge the host message.

## `globalThis.MagicUIEvents`

`MagicUIEvents` is not injected by the runtime. It is installed by the
optional helper file shipped at:

```text
Plugins/MagicUI/Content/Web/EventsTest/magicui-events.js
```

Copy that file into the imported HTML source folder and load it before your
application script:

```html
<script src="magicui-events.js"></script>
<script src="app.js" defer></script>
```

Conceptual declaration:

```typescript
type MagicUIEventListener = (
  payload: JSONValue,
  eventName: string
) => void;

type MagicUIUnsubscribe = () => boolean;

interface MagicUIEventChannel {
  emit(eventName: string, payload?: JSONValue): true;
  addListener(
    eventName: string,
    listener: MagicUIEventListener
  ): MagicUIUnsubscribe;
  once(
    eventName: string,
    listener: MagicUIEventListener
  ): MagicUIUnsubscribe;
  removeListener(
    eventName: string,
    listener: MagicUIEventListener
  ): boolean;
  clearListeners(eventName?: string): void;
}

interface MagicUIEventsFacade extends MagicUIEventChannel {
  readonly protocolVersion: 1;
  readonly gameplayTags: Readonly<MagicUIEventChannel>;
}

declare const MagicUIEvents: Readonly<MagicUIEventsFacade>;
```

### `MagicUIEvents.protocolVersion`

The helper's numeric event-envelope version. Its current value is `1`.

```javascript
console.log(MagicUIEvents.protocolVersion); // 1
```

Do not use this value to invent a different envelope. Keep the helper file from
the same plugin version as the Unreal plugin.

### `MagicUIEvents.emit(eventName, payload?)`

Sends a simple named event from JavaScript to a **MagicUI Event** component.

```javascript
MagicUIEvents.emit("inventory.open", { slot: 3 });
MagicUIEvents.emit("menu.closed"); // payload becomes null
```

| Detail | Behavior |
| --- | --- |
| Name matching | Case-sensitive |
| Missing payload | JSON `null` |
| Success return | `true` after validation and a synchronous call to the raw bridge |
| Failure | Throws for an invalid name, missing bridge, serialization failure, or synchronous native admission failure |
| Unreal receiver | **On Event Received(Event Name, Json Payload)** |

Its `true` return is not proof that a named-event component or Blueprint
handler processed the event, and it is not an application acknowledgement.

### `MagicUIEvents.addListener(eventName, listener)`

Listens for a simple event emitted by a **MagicUI Event** component.

```javascript
function onInventoryUpdated(payload, eventName) {
  console.log(eventName, payload);
}

const unsubscribe = MagicUIEvents.addListener(
  "inventory.updated",
  onInventoryUpdated
);
```

The callback receives the parsed `payload` first and the event name second.
The method returns an idempotent unsubscribe function:

```javascript
unsubscribe(); // true if it removed the subscription
unsubscribe(); // false; it was already removed
```

### `MagicUIEvents.once(eventName, listener)`

Registers a listener that removes itself before its first callback runs.

```javascript
MagicUIEvents.once("dialog.initialized", (payload, eventName) => {
  console.log(eventName, payload);
});
```

It returns the same kind of unsubscribe function as `addListener`, so the
one-shot listener can also be cancelled before it fires.

### `MagicUIEvents.removeListener(eventName, listener)`

Removes the exact function previously registered for the exact event name.

```javascript
function onUpdate(payload) {
  console.log(payload);
}

MagicUIEvents.addListener("inventory.updated", onUpdate);
const removed = MagicUIEvents.removeListener("inventory.updated", onUpdate);
```

Returns `true` when the listener existed and was removed; otherwise returns
`false`. An anonymous function cannot be removed later unless its reference was
saved.

### `MagicUIEvents.clearListeners(eventName?)`

With a name, removes every simple listener for that exact event:

```javascript
MagicUIEvents.clearListeners("inventory.updated");
```

With no argument, removes every simple-channel listener:

```javascript
MagicUIEvents.clearListeners();
```

It does not clear Gameplay Tag listeners. Use the method on
`MagicUIEvents.gameplayTags` for that channel.

## `MagicUIEvents.gameplayTags`

The `gameplayTags` object has the same five methods as the simple channel. It
uses the `gameplay-tag` envelope channel and is paired with the **MagicUI
Gameplay Tag Event** component.

```javascript
MagicUIEvents.gameplayTags.emit(
  "UI.Inventory.Item.Selected",
  { itemId: "potion-small" }
);

const unsubscribe = MagicUIEvents.gameplayTags.addListener(
  "UI.Inventory.Item.Confirmed",
  (payload, eventName) => {
    console.log(eventName, payload);
  }
);
```

Gameplay Tag listener lookup is case-insensitive. Unreal accepts page tags only
when an exact registered tag is selected in the component's **Gameplay Tags To
Listen For** container. An empty container accepts none.

See [Use Gameplay Tag events](../guides/gameplay-tag-events.md) for the Unreal
setup.

## Event-name validation

Both helper channels require an event name that is:

- a string;
- non-empty;
- no more than 256 UTF-16 code units (`eventName.length`); and
- free of control characters from U+0000 through U+001F.

An invalid name throws `TypeError` in JavaScript. Simple names are
case-sensitive; Gameplay Tag listener names are case-insensitive.

## Listener scheduling and errors

The helper takes a stable snapshot of matching listeners, then invokes them in
a JavaScript microtask. As a result:

- a native `magic-message` dispatch does not directly re-enter application
  callbacks;
- adding or removing a listener during a callback does not alter the current
  snapshot;
- one listener throwing does not prevent the remaining listeners from running;
  and
- a thrown listener error is reported asynchronously with `setTimeout`.

The helper records its page-owned router on the Window. When a new document is
loaded into a retained Window, loading the helper again removes the previous
router and clears its listeners before installing the replacement.

## Protocol envelope

The helper serializes this envelope:

```typescript
interface MagicUIEventEnvelope {
  protocol: "magicui.event";
  version: 1;
  channel: "event" | "gameplay-tag";
  event: string;
  payload: JSONValue;
}
```

Example:

```json
{
  "protocol": "magicui.event",
  "version": 1,
  "channel": "event",
  "event": "inventory.open",
  "payload": {
    "slot": 3
  }
}
```

The Unreal component returns only `event` and a reserialized compact
`payload` to Blueprint. A missing envelope payload is normalized to JSON
`null`.

Use the helper rather than manually building envelopes. Wrong versions,
missing fields, malformed JSON, and the wrong channel are invalid or ignored
by the corresponding component.

## Unreal-side API map

| Direction/use | Blueprint node or event | Meaning |
| --- | --- | --- |
| Page → Unreal, raw | **On Page Message** | Complete serialized JSON value |
| Unreal → page, raw | **Post Message to Page** | Accept a strict JSON value string for async delivery |
| Page → Unreal, named | **On Event Received** | Simple event name + compact JSON payload |
| Unreal → page, named | **Emit Event** | Prepare a named event asynchronously |
| Event failure | **On Event Error** | Validation, readiness, size, source, or queue error |
| Page → Unreal, tag | **On Gameplay Tag Event Received** | Typed registered tag + compact JSON payload |
| Unreal → page, tag | **Emit Gameplay Tag Event** | Emit a typed registered tag |
| One-off script | **Evaluate JavaScript Async** | Return request ID immediately |
| Script result | **On JavaScript Result** | Request ID, success, JSON result, exception |

All of these Blueprint delegates run on Unreal's game thread.

## `Evaluate JavaScript Async`

JavaScript evaluation is a Blueprint/C++ component operation rather than a
page global, but its result follows the same JSON contract.

```text
Evaluate JavaScript Async(Script) → Request Id
On JavaScript Result(Request Id, Success, Json Result, Exception)
```

Rules:

- call it after **On Page Loaded**;
- request ID `0` means the call was rejected;
- correlate every result with its request ID;
- successful values are returned as serialized JSON;
- `undefined` becomes `null`;
- thrown errors and non-JSON-compatible return values set **Success = false**;
- plain values and promises that settle in the bounded evaluation turn are
  supported; arbitrary asynchronous promises are not waited on; and
- MagicUI has no synchronous JavaScript execution API.

Example script:

```javascript
({ title: document.title, focused: document.hasFocus() })
```

Possible successful **Json Result**:

```json
{"title":"Inventory","focused":true}
```

## Limits and delivery guarantees

For **MagicUI Event** and **MagicUI Gameplay Tag Event**:

- **Max Event Bytes** defaults to 65,536 bytes and is clamped to
  1,024–65,536;
- the limit covers the complete UTF-8 envelope;
- each component keeps at most 64 active/queued preparation items per
  direction;
- one item per direction is prepared at a time, preserving that component's
  order;
- runtime transport queues impose additional limits;
- a Blueprint emit's `true` output means accepted for component preparation,
  while a JavaScript helper emit's `true` only means it called the raw bridge
  without a synchronous exception;
- delivery is at-most-once, with no automatic retry or acknowledgement; and
- separate components and raw/navigation/evaluation APIs have independent
  queues, so cross-API order is not guaranteed.

For reliable application confirmation, include a request ID in the JSON
payload and send a separate response event.

## Quick examples

### Page action to Blueprint

```javascript
MagicUIEvents.emit("menu.option.selected", {
  option: "graphics",
  index: 1
});
```

Blueprint receives:

```text
Event Name   = menu.option.selected
Json Payload = {"option":"graphics","index":1}
```

### Blueprint state to page

Blueprint emits:

```text
Event Name   = player.stats.changed
Json Payload = {"health":85,"armor":20}
```

Page listener:

```javascript
MagicUIEvents.addListener("player.stats.changed", (stats) => {
  document.querySelector("#health").textContent = String(stats.health);
  document.querySelector("#armor").textContent = String(stats.armor);
});
```

### Raw page message

```javascript
magic.postMessage({ type: "diagnostic", text: "Page mounted" });
```

### Raw host message listener

```javascript
window.addEventListener("magic-message", ({ detail }) => {
  console.log("Unreal sent", detail);
});
```

## Related guides

- [Raw bridge tutorial](../guides/javascript-bridge.md)
- [MagicUI Event tutorial](../guides/magicui-events.md)
- [Gameplay Tag event tutorial](../guides/gameplay-tag-events.md)
