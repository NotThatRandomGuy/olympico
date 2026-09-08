# Olympico — agent notes

Single-file static game: everything lives in `index.html` (CSS + one `<script type="module">`).
Dependencies are CDN-only: Three.js + cannon-es (import map), PeerJS (signaling/WebRTC), mqtt.js (LAN session browser).

## Verify

- Syntax check the module script: extract the `<script type="module">` body to a `.mjs` file and run `node --check`.
- Runtime smoke test: serve the folder with any static server (e.g. `npx serve .` or a tiny Node http server) and open
  `index.html` in Chrome. `window.olympico` exposes `{ game, network, discovery, car, ballBody, input, remoteCars, ghostCars, botControllers, startGame }`
  for scripted checks; two iframes of the page in one harness can host + join a room end-to-end (host: `network.host(name)`,
  guest: `network.join(name, code)`, then `network.launch()` on the host).
- Headless Chrome with `--headless=new --use-angle=swiftshader --enable-unsafe-swiftshader --enable-logging=stderr` prints
  page console output, which is enough to assert on the checks above.

## Netcode model (multiplayer)

- Host is authoritative for clock, score, ball, bots and boost meters. Snapshots go out at 30 Hz with a `reset` sequence
  number; guests snap their own car when it changes and echo it back as `ack` so the host ignores stale guest state after kickoffs.
- Guests simulate their own car and a local dynamic ball (for immediate contact feedback); the ball is blended toward the host
  each snapshot. Guest ball contacts also send a `ball-hit` hint the host only applies if its own physics missed the contact.
- Remote cars on the host are full physics bodies driven by guest inputs and reconciled toward the guest's reported state.
  Other players on a guest are non-physical ghosts interpolated/extrapolated every frame.
- Session browser: hosts heartbeat to a public MQTT broker on topic `olympico/v1/<sha256(public ip)>/<code>`; browsers on
  the same public IP see them. Falls back to a `public` scope if IP lookup fails.
