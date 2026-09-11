# MX Creative Console Script Lab

The interactive game scripting workbench is in [test/index.html](test/index.html). It loads the packaged WebHID client, runs user JavaScript in a Web Worker, mirrors display commands to a 3x3 keypad emulator, and can upload serialized frames to an attached MX Creative Keypad.

## Run locally

Serve the repository over HTTP rather than opening the HTML file directly. The server must serve `.mjs` files with a JavaScript MIME type.

```bash
python -m http.server 8000
```

Open `http://localhost:8000/test/index.html` in Chrome or Edge. WebHID connection and File System Access prompts require a user gesture. The Save and Load buttons use the native file picker when available and fall back to browser downloads/uploads.

## Script lifecycle

Scripts provide `init(state)`, `update(dt, input, state)`, and `draw(display, state)`. Script input keys use zero-based indices `0` through `8`, matching the raw device layer. The packaged client's HID control IDs `1` through `9` are converted to those same indices at the event boundary. The runtime calls logic at approximately 30 Hz and emits display frames at the keypad's 10 FPS limit. Drawing is command-based so the script worker never receives DOM, canvas, or WebHID objects.

```js
function draw(display, state) {
  display.keys.forEach((key) => key.fill('#101820'));
  display.keys[4].fillRect(state.x, state.y, 10, 10, '#b8ee54');
}
```
