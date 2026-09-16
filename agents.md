# MX Creative Console WebHID Developer Guide

This document is the implementation guide for coding agents and human developers integrating the packaged Logitech MX Creative Console WebHID client into a web application.

The client communicates directly with supported Logitech devices through the WebHID API. It does not provide a server-side transport, a Bluetooth transport, or a generic HID abstraction.

## What the library provides

The package exports exactly four public names:

- `MXCreativeConsoleClient`: high-level device discovery, connection, input, brightness, roller, and keypad-display API.
- `KEYPAD_KEYS`: metadata for the nine keypad keys and the `PREV`/`NEXT` paging keys.
- `DIALPAD_KEYS`: metadata for the four dialpad controls.
- `ReportingMode`: roller reporting mode constants, `Native` and `Diverted`.

The current implementation recognizes these Logitech devices during the WebHID permission request:

| Device | Vendor ID | Product ID | Library role |
| --- | ---: | ---: | --- |
| MX Creative Keypad | `0x046d` | `0xc354` | `keypad` |
| MX Creative Dialpad | `0x046d` | `0xbc00` | `dialpad` |

Role classification is feature-based. A device with contextual-display support is classified as `keypad`; otherwise a device with multi-roller support is classified as `dialpad`. A feature-compatible device that exposes only another supported feature is classified as `keypad` by the fallback rule.

## Runtime requirements

- A browser with WebHID support, such as Chrome 89+, Edge 89+, or Opera 75+.
- An HTTP or HTTPS origin. A page opened directly from an arbitrary `file://` URL cannot use WebHID.
- A user gesture for `navigator.hid.requestDevice()`. Put `requestDeviceAccessAndScan()` directly in a click, pointer, or keyboard handler.
- A browser context with `navigator.hid` available. The client does not polyfill WebHID.
- The browser must be able to decode images with `createImageBitmap`, `Image`, and canvas APIs for contextual-display uploads.

WebHID permission is granted per origin. The library can rediscover already-authorized devices without opening the browser picker, but the first permission request must be initiated by the user.

## Library files

- `js/mx-creative-console.min.mjs`: named ES module bundle.
- `js/mx-creative-console.min.js`: named exports attached to the `window.MXCreativeConsole` IIFE namespace.

## Basic ES module integration

```html
<button id="connect" type="button">Connect MX Creative Console</button>
<pre id="status" aria-live="polite"></pre>

<script type="module">
  import { MXCreativeConsoleClient } from './js/mx-creative-console.min.mjs';

  const client = new MXCreativeConsoleClient({ debug: true });
  const status = document.querySelector('#status');
  const connectButton = document.querySelector('#connect');

  client.on('status', ({ message, isError }) => {
    status.textContent = message;
    status.dataset.error = String(isError);
  });

  connectButton.addEventListener('click', async () => {
    connectButton.disabled = true;
    try {
      const devices = await client.requestDeviceAccessAndScan();
      const keypad = devices.find((device) => device.role === 'keypad');
      if (!keypad) {
        throw new Error('No compatible MX Creative keypad was selected.');
      }
      await client.connectAvailableDevice(keypad.index);
    } catch (error) {
      status.textContent = error instanceof Error ? error.message : String(error);
      connectButton.disabled = false;
    }
  });

  window.addEventListener('beforeunload', () => {
    void client.disconnectAll();
  });
</script>
```

`requestDeviceAccessAndScan()` both opens the permission picker and scans the authorized devices. The returned array contains the devices discovered by the client, not raw `HIDDevice` objects alone. Do not assume the first returned device is a keypad; inspect `role`, capability flags, and the friendly name.

For applications that manage permission and scanning separately, the client also exposes:

```js
await client.scanAuthorizedDevices(); // no picker; scans navigator.hid.getDevices()
const entries = client.getAvailableDevices(); // current metadata snapshot
```

`scanAuthorizedDevices()` emits `devicesChanged` and returns the fresh metadata array. It is useful after a device is authorized elsewhere in the application or after the browser's HID device list changes.

## Classic browser-script integration

```html
<script src="./js/mx-creative-console.min.js"></script>
<script>
  const {
    MXCreativeConsoleClient,
    KEYPAD_KEYS,
    DIALPAD_KEYS,
    ReportingMode
  } = window.MXCreativeConsole;

  const client = new MXCreativeConsoleClient();
  console.log(KEYPAD_KEYS, DIALPAD_KEYS, ReportingMode);
</script>
```

The IIFE namespace is `window.MXCreativeConsole`. The ES module and IIFE expose the same four public names.

## Client construction and event listeners

```js
const client = new MXCreativeConsoleClient({ debug: false });

function onStatus(payload) {
  console.log(payload.message);
}

client.on('status', onStatus);
client.off('status', onStatus);
```

The constructor accepts an optional object. The supported option is `debug`; truthy values enable diagnostic messages prefixed with `[mx-webhid]`. `on(eventName, handler)` adds a handler to the event, and `off(eventName, handler)` removes that exact function reference. Listener exceptions are caught and logged by the client so one faulty listener does not stop other listeners.

## Recommended connection lifecycle

### 1. Request permission and scan

```js
const devices = await client.requestDeviceAccessAndScan();
```

This method:

1. Verifies `navigator.hid` exists.
2. Requests Logitech devices using the two supported vendor/product filters.
3. Scans all previously authorized Logitech devices.
4. Probes support for multi-roller, special keys, brightness, and contextual display features.
5. Attempts to read the device friendly name through HID++ feature `0x0007`.
6. Emits `devicesChanged` and returns the available-device metadata.

If there are no compatible devices, the method returns an empty array and emits an error status. Permission cancellation or a WebHID error rejects the promise.

### 2. Inspect device metadata

Each returned device entry has this shape:

```js
{
  index: 0,
  role: 'keypad', // 'keypad' or 'dialpad'
  friendlyName: '...',
  nameSource: '0x0007', // or 'productName'
  productName: '...',
  productId: 0xc354,
  vendorId: 0x046d,
  device: HIDDevice,
  supportsMultiRoller: false,
  supportsSpecialKeys: true,
  supportsBrightness: true,
  supportsContextualDisplay: true,
  supportsAnyFeature: true
}
```

`index` is the index used by `connectAvailableDevice()`. Preserve the returned entry or re-read the list rather than inventing an index from the physical device order.

### 3. Connect by index

```js
const state = await client.connectAvailableDevice(device.index);
```

The client disconnects an existing connection for the same role before opening the selected device. Keypad initialization includes special-key reporting, contextual display, and brightness. Dialpad initialization includes rollers and special keys. Unsupported optional features are disabled individually; a device can still connect if only some feature probes succeed.

For advanced integrations, `connectDevice(role, deviceEntry)` is also available, but it expects a metadata entry in the same shape produced by scanning. Prefer `connectAvailableDevice()` for normal application code.

### 4. Disconnect

```js
await client.disconnectRole('keypad');
await client.disconnectRole('dialpad');
await client.disconnectAll();
```

Disconnect restores the original special-key reporting settings. Dialpad disconnect also attempts to return every roller to `ReportingMode.Native`. Call `disconnectAll()` during application teardown or page unload. Because browser unload handling may terminate asynchronous work, applications should also disconnect when their own device-management view is closed.

## State and capability inspection

```js
const state = client.getConnectionState();
console.log(state);
```

The state object has this shape:

```js
{
  keypadConnected: true,
  dialpadConnected: false,
  features: {
    keypad: {
      specialKeys: true,
      brightness: true,
      contextualDisplay: true
    },
    dialpad: {
      specialKeys: false,
      multiRoller: false
    }
  },
  brightnessInfo: {
    maxBrightness: 100,
    steps: 100,
    capabilities: 0,
    minBrightness: 0
  },
  contextualDisplayCaps: { /* see contextual display section */ },
  contextualDisplayInfo: { /* see contextual display section */ },
  rollerCapabilities: [],
  rollerEventCount: 0
}
```

Feature initialization is best-effort. Check the corresponding capability flag before rendering controls, and handle method rejection anyway because a device can disconnect after initialization.

## Key metadata and key events

### Key tables

```js
KEYPAD_KEYS
// [{ controlId: 0x0001, label: '1' }, ...,
//  { controlId: 0x0009, label: '9' },
//  { controlId: 0x01a1, label: 'PREV' },
//  { controlId: 0x01a2, label: 'NEXT' }]

DIALPAD_KEYS
// [
//   { controlId: 0x0053, label: 'D1' },
//   { controlId: 0x0056, label: 'D2' },
//   { controlId: 0x0059, label: 'D3' },
//   { controlId: 0x005a, label: 'D4' }
// ]
```

Control IDs are numeric HID++ control IDs. Use the exported tables instead of duplicating these values in UI code.

### Keypad

```js
client.on('keypadKeysChanged', ({ source, activeControlIds, activeLabels }) => {
  console.log(source, activeControlIds, activeLabels);
});
```

The event payload is:

```js
{
  source: 'raw' | 'diverted',
  activeControlIds: number[],
  activeLabels: string[]
}
```

The client merges normal raw VLP reports and diverted special-key reports into one active-key snapshot. The array represents the current state, not only the key that changed. A release can therefore produce an empty array. `activeLabels` only includes keys present in `KEYPAD_KEYS`; use `activeControlIds` when building behavior for additional controls.

### Dialpad

```js
client.on('dialpadKeysChanged', ({ source, activeControlIds, activeLabels }) => {
  console.log(source, activeControlIds, activeLabels);
});
```

The payload has the same shape as the keypad event, but only the controls in `DIALPAD_KEYS` are retained. Normal reports may use low-byte IDs; the client normalizes those against the exported full control IDs.

## Dialpad rollers

Roller support is initialized automatically when a connected device is classified as a dialpad and exposes feature `0x4610`.

### Capabilities

```js
const state = client.getConnectionState();
for (const roller of state.rollerCapabilities) {
  console.log({
    id: roller.rollerId,
    incrementsPerRotation: roller.incrementsPerRotation,
    incrementsPerRatchet: roller.incrementsPerRatchet,
    lightbarId: roller.lightbarId,
    timestampReport: roller.timestampReport,
    ratcheted: roller.isRatcheted(),
    hasLightbar: roller.hasLightbar(),
    supportsTimestamp: roller.supportsTimestamp()
  });
}
```

Each capability object contains `rollerId`, `incrementsPerRotation`, `incrementsPerRatchet`, `lightbarId`, and `timestampReport`, plus the helper methods `isRatcheted()`, `hasLightbar()`, and `supportsTimestamp()`.

### Reporting mode

```js
await client.setRollerDiverted(true);  // use diverted HID++ rotation events
await client.setRollerDiverted(false); // restore native reporting
```

The method applies the selected mode to every discovered roller and emits:

```js
client.on('rollerModeChanged', ({ mode, modeName, diverted }) => {
  // mode is ReportingMode.Native or ReportingMode.Diverted
  console.log(mode, modeName, diverted);
});
```

The exported constants are:

```js
ReportingMode.Native   // 0
ReportingMode.Diverted // 1
```

`setRollerDiverted()` rejects when no roller feature is available.

### Rotation events and position snapshots

```js
client.on('rollerEvent', ({ rollerId, delta, timestamp, eventCount, snapshot }) => {
  const changed = snapshot.find((roller) => roller.isChanged);
  console.log(rollerId, delta, timestamp, changed?.progress);
});

client.clearRollerEvents();
```

The `rollerEvent` payload is:

```js
{
  rollerId: number,
  delta: number,       // signed movement increment
  timestamp: number,   // 0 when the report has no timestamp
  eventCount: number,
  snapshot: [
    {
      rollerId: number,
      available: boolean,
      isChanged: boolean,
      directionClass: 'up' | 'down' | 'idle',
      progress: number, // normalized 0..1 position
      position: number,
      incrementsPerRotation: number
    },
    // a second entry is included for the two-roller UI model
  ]
}
```

The snapshot always contains entries for roller IDs `0` and `1`; unavailable entries have `available: false`. Position wraps within `incrementsPerRotation`. `clearRollerEvents()` resets the event counter and tracked positions, then emits `rollerCleared` with `{ eventCount: 0 }`.

## Keypad brightness

Brightness is initialized automatically for connected keypad devices exposing feature `0x8040`.

```js
client.on('brightnessChanged', ({ raw, percent }) => {
  brightnessSlider.value = percent;
});

const state = client.getConnectionState();
console.log(state.brightnessInfo);
await client.refreshBrightness();
await client.setBrightnessPercent(65);
```

`setBrightnessPercent(percent)` clamps and rounds the requested value to `0..100`, converts it to the device's raw range, writes it, then refreshes and returns `{ raw, percent }`. `refreshBrightness()` returns `null` if brightness is unavailable; otherwise it returns the same object and emits `brightnessChanged`.

`brightnessInfo` contains:

```js
{
  maxBrightness: number,
  minBrightness: number,
  steps: number,
  capabilities: number
}
```

Both brightness methods can reject if the device disconnects or the feature is unavailable. Do not use the raw value as a percentage; use the returned `percent` value for UI.

## Contextual display images

Contextual display support is initialized automatically for connected keypad devices exposing the MX-specific feature `0x19a1` and output report `0x14`.

### Capabilities and layout

```js
const state = client.getConnectionState();
console.log(state.contextualDisplayCaps);
console.log(state.contextualDisplayInfo);
```

For the MX Creative Keypad implementation, the reported static capabilities are:

```js
{
  deviceScreenCount: 1,
  maxImageSize: 300 * 1024,
  maxImageFPS: 10,
  deferrableDisplayUpdate: true,
  rgb565: false,
  rgb888: true,
  jpeg: true,
  calibrated: false,
  origin: 0
}
```

The display information describes one `457 x 440` logical display containing nine `118 x 118` buttons. Button locations are:

```js
{
  x: 23 + column * 158,
  y: 6 + row * 158,
  w: 118,
  h: 118
}
```

where `row` and `column` are zero-based. Use `contextualDisplayInfo.buttons` rather than hardcoding coordinates if your UI needs to mirror the device.

### Upload one key

```js
const input = document.querySelector('#image');
await client.uploadContextualDisplayImage(input.files[0], {
  mode: 'single',
  keyNumber: 1
});
```

`file` must be a browser `File` or image `Blob` with an `image/*` MIME type. The client decodes it, scales it to the target display area, and encodes it as JPEG when possible, falling back to RGB888 when required by size/capability limits.

`keyNumber` is one-based and must be `1..9`. The method emits:

```js
client.on('imageUploadComplete', ({ mode, keyNumber }) => {
  console.log(`Uploaded ${mode} image to key ${keyNumber}`);
});
```

### Upload one image to all nine keys

```js
await client.uploadContextualDisplayImage(file, { mode: 'all' });
```

The image is encoded for the button dimensions and sent nine times. Updates are deferred for the first eight writes and committed on the ninth. Status events report progress as `Uploading image to key N of 9…`. The completion payload is `{ mode: 'all' }`.

### Upload a full-display image

```js
await client.uploadContextualDisplayImage(file, { mode: 'full' });
```

When `options.mode` is omitted, the mode defaults to `'single'`. Any other truthy mode besides `'single'` and `'all'` follows the full-display path, so use `'full'` explicitly. The image is scaled to the full logical display resolution and uploaded at location `{ x: 0, y: 0, w: 457, h: 440 }`. Completion emits `{ mode: 'full' }`.

### Image upload errors

Uploads reject when:

- no contextual-display feature is available;
- the file is missing or is not an image;
- the image cannot be decoded;
- `keyNumber` is outside `1..9`;
- the device reports fewer than nine buttons for `mode: 'all'`;
- canvas encoding cannot fit a supported JPEG or RGB888 payload under the device limit; or
- the HID device disconnects during transmission.

The implementation sends large report `0x14` frames sequentially and does not wait for an acknowledgment between individual frames. Avoid starting multiple uploads at once; serialize application-level uploads to prevent interleaving packets.

## Event reference

The high-level client emits these events:

| Event | Payload | When |
| --- | --- | --- |
| `status` | `{ message: string, isError: boolean }` | Progress, informational status, or recoverable error text. |
| `devicesChanged` | `DeviceEntry[]` | A scan completes. |
| `roleConnected` | `{ role, deviceEntry }` | A keypad or dialpad finishes initialization. |
| `roleDisconnected` | `{ role }` | A role is disconnected. |
| `stateChanged` | `getConnectionState()` result | Connection or feature state changes. |
| `keypadKeysChanged` | `{ source, activeControlIds, activeLabels }` | Current keypad active-key set changes. |
| `dialpadKeysChanged` | `{ source, activeControlIds, activeLabels }` | Current dialpad control set changes. |
| `brightnessChanged` | `{ raw, percent }` | Brightness is refreshed or changed. |
| `rollerModeChanged` | `{ mode, modeName, diverted }` | All rollers change reporting mode. |
| `rollerEvent` | `{ rollerId, delta, timestamp, eventCount, snapshot }` | A diverted roller rotation is received. |
| `rollerCleared` | `{ eventCount: 0 }` | Roller positions and event count are reset. |
| `imageUploadComplete` | `{ mode, keyNumber? }` | A contextual-display upload finishes. |

The client does not emit a separate `error` event. Subscribe to `status` and inspect `isError`, and also catch rejected promises from every asynchronous operation.

## Minimal keypad application pattern

```js
import {
  MXCreativeConsoleClient,
  KEYPAD_KEYS
} from './js/mx-creative-console.min.mjs';

const client = new MXCreativeConsoleClient();
let connectedKeypad = null;

client.on('keypadKeysChanged', ({ activeControlIds }) => {
  const active = new Set(activeControlIds);
  for (const key of KEYPAD_KEYS) {
    document.querySelector(`[data-control-id="${key.controlId}"]`)
      ?.toggleAttribute('data-active', active.has(key.controlId));
  }
});

async function connectKeypad() {
  const entries = await client.requestDeviceAccessAndScan();
  connectedKeypad = entries.find((entry) => entry.role === 'keypad');
  if (!connectedKeypad) {
    throw new Error('Select an MX Creative keypad.');
  }
  return client.connectAvailableDevice(connectedKeypad.index);
}

async function updateKey(file, keyNumber) {
  await client.uploadContextualDisplayImage(file, {
    mode: 'single',
    keyNumber
  });
}
```

Keep the connect call inside the button handler:

```js
connectButton.addEventListener('click', () => {
  connectKeypad().catch(renderError);
});
```

## Error handling and operational guidance

- Treat all device operations as asynchronous and catch every rejection.
- Disable or hide controls when the matching `getConnectionState().features` flag is false.
- A successful scan does not guarantee that the device remains connected. USB removal and browser permission changes can invalidate a later operation.
- Do not retain a stale device index after rescanning; use the latest returned metadata.
- Do not call `requestDeviceAccessAndScan()` automatically at page load. Browsers require a user activation for the picker.
- Call `disconnectAll()` when leaving the device-management workflow. This restores reporting modes and removes input listeners.
- Revoke application-created object URLs used for local image previews with `URL.revokeObjectURL()`.
- Serialize contextual-display uploads. The low-level image transport sends sequential frames for one image but does not coordinate concurrent application calls.
- Update UI from `status` and `stateChanged` instead of assuming that initialization of every optional feature succeeded.
