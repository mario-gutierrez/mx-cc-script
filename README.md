# MX Keypad Script Lab

The interactive game scripting workbench is in [index.html](./index.html). It loads the packaged WebHID client, runs user JavaScript in a Web Worker, mirrors display commands to a 3x3 keypad emulator, and can upload serialized frames to an attached MX Keypad.

![MX Keypad Script Lab](assets/mx-keypad-script.gif)

## Disclaimer

This is an experimental project, it is not officially supported by Logitech.

## Run locally

Serve the repository over HTTP rather than opening the HTML file directly. The server must serve `.mjs` files with a JavaScript MIME type.

```bash
python -c "import http.server,mimetypes; mimetypes.add_type('text/javascript','.mjs'); http.server.test(HandlerClass=http.server.SimpleHTTPRequestHandler,port=8000)"
```

Open `http://localhost:8000` in Chrome or Edge. WebHID connection and File System Access prompts require a user gesture. The Save and Load buttons use the native file picker when available and fall back to browser downloads/uploads.

## Sample scripts

The editor's tab strip switches between three bundled scripts:

- **Starter** — a minimal `init`/`update`/`draw` example that moves a square around the center key with keys `1`, `3`, `5`, `7`.
- **Space Invaders** — a full game spanning all nine keys as one 354x354 canvas. Move with keys `6`/`8`, fire with `NEXT`, restart with `PREV` after a loss.
- **Weather** — fetches current conditions for the browser's geolocated position from [Open-Meteo](https://open-meteo.com) (no API key required), auto-refreshing every 10 minutes or immediately on `PREV`. Demonstrates `fetch`, `getLocation`, and the `fillCircle`/`arc`/`fillText`/`drawBitmap` primitives to draw a weather icon, a countdown ring, and text.

## Script lifecycle

Scripts provide `init(state)`, `update(dt, input, state)`, and `draw(display, state)`. Script input keys use zero-based indices `0` through `8`, matching the raw device layer. The packaged client's HID control IDs `1` through `9` are converted to those same indices at the event boundary. The worker tick, browser preview, and hardware display uploads all run at a matched 10 Hz — the keypad's advertised upload limit — so frames are never produced faster than they can be shown, and superseded frames do not build up in the WebHID queue. Input state is reset at run and stop boundaries so a new game cannot inherit a previously latched key snapshot. Drawing is command-based so the script worker never receives DOM, canvas, or WebHID objects.

```js
function draw(display, state) {
  display.keys.forEach((key) => key.fill('#101820'));
  display.keys[4].fillRect(state.x, state.y, 10, 10, '#b8ee54');
}
```

## Drawing API

Each entry in `display.keys` (one per physical key, index `0`-`8`) supports:

| Call | Description |
| --- | --- |
| `fill(color)` | Fills the entire key with a solid color. |
| `fillRect(x, y, width, height, color)` | Fills an axis-aligned rectangle. |
| `fillCircle(x, y, radius, color)` | Fills a circle. |
| `arc(x, y, radius, startAngle, endAngle, color)` | Fills a pie slice; useful for radial gauges and countdown rings. |
| `fillText(text, x, y, color, size)` | Draws text using the canvas's native font rendering. |
| `drawBitmap(rows, x, y, pixelSize, color)` | Draws a small icon from an array of `'0'`/`'1'` strings, one filled `pixelSize` square per `'1'`. |

Coordinates are local to each 118x118 key canvas.

## Network and location access

Scripts run inside a real Web Worker, so `fetch(url)` works directly for calling external HTTP APIs (subject to the target's CORS policy). `navigator.geolocation` is not available inside a Worker, so the engine exposes a `getLocation()` global instead: it forwards the request to the host page, which calls `navigator.geolocation.getCurrentPosition` and returns `{ latitude, longitude }` back to the script. Both `fetch` and `getLocation()` return Promises; since `init`/`update`/`draw` are never awaited by the engine, wrap asynchronous work in `try`/`catch` and store results or errors on `state` for `draw` to render, as the Weather script does.
