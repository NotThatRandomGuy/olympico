# Olympico

**Arcade vehicle football in the browser.** Drive, boost, jump and flip a rocket-powered car to smash a giant ball into the other team's net — *Rocket League*-style, but running entirely as a single static HTML page. Play against bots, practise in the training arena, or host a peer-to-peer match for friends on your network.

- Zero install, zero build step: open `index.html` or drop the folder on any static host
- Full arcade car kit: acceleration, drifting, jumps, double jumps, dodges/flips, aerial control and a boost meter
- Horizon-locked chase camera that stays level no matter how the car tumbles, with a toggleable ball cam
- Single player 1v1 → 4v4 with pursuit/defend bots, plus a free-play Training Grounds
- Peer-to-peer multiplayer over WebRTC with room codes and an auto-discovering session browser
- Three body styles, three wheel shapes, blue/orange team colourways — cosmetics only, everyone drives the same hitbox

---

## Quick start

**Play locally**

```bash
git clone https://github.com/NotThatRandomGuy/olympico.git
cd olympico
# any static server works; for example:
npx serve .
```

Then open the printed URL (or simply double-click `index.html` — it also runs from `file://`, though multiplayer needs `http(s)://`).

**Deploy**

There is nothing to build. Point Netlify, GitHub Pages, Vercel, Cloudflare Pages or any web server at the repository root so `index.html` is served. All libraries are loaded from public CDNs at runtime.

---

## How to play

### Objective

Score more goals than the other team before the five-minute clock runs out. If the score is tied at full time, the match goes to **overtime**: the next goal wins. Every goal fires a shockwave out of the net that blasts nearby cars backwards, then the pitch resets to a kickoff formation a couple of seconds later — no replays, straight back into it.

### Controls

| Action | Keyboard / mouse | Gamepad |
| --- | --- | --- |
| Accelerate / reverse | `W` / `S` (or `↑` / `↓`) | Left stick up / down |
| Steer | `A` / `D` (or `←` / `→`) | Left stick left / right |
| Boost | `Shift` (hold) | `RB` |
| Jump · double jump · dodge | `J`, `Numpad 0`, or left mouse click | `A` |
| Powerslide / handbrake | `Space` (hold) | `B` |
| Toggle ball cam | tap `Space`, or `C` | Left stick click |
| Air roll | `Space` + `A` / `D` | Right stick |
| Reset ball *(Training only)* | `R` | — |
| Pause / menu | `Esc` | — |

### Car handling tips

- **Boost** drains while held and refills from the glowing orange pads dotted around the pitch (each pad gives a chunk of boost and recharges after five seconds). Training Grounds gives you infinite boost.
- **Powersliding** (hold `Space` while turning) lets the rear break loose for tight, drifting turns and sharper rotation.
- **Jump** once from the ground, then press jump again in the air within 1.5 seconds:
  - with no direction held you get a **double jump**;
  - with a direction held you **dodge** that way — forward flips, back flips and side flips add a burst of speed and are the fastest way to poke the ball.
- While airborne, `W`/`S` pitch the nose, `A`/`D` yaw, and `Space`+`A`/`D` roll. Aim your car, feather the boost and you can fly.
- **Wall riding**: drive up the arena walls and ceiling — the car sticks as long as its wheels touch a surface.
- Landed on your roof? Hold still for a moment or press jump and the car will flip itself back onto its wheels.
- **Ball cam** keeps the camera pointed at the ball wherever you are facing; when the ball is off-screen an arrow at the edge of the HUD shows where it is.

### Modes

| Mode | What it is |
| --- | --- |
| **Quick Match** | Instant 1v1 against a bot; you play Blue. |
| **Single Player** | Pick 1v1, 2v2, 3v3 or 4v4 and your team; bots fill every other seat, chasing the ball and dropping back to defend. |
| **Direct Connect** | Peer-to-peer multiplayer — host a room or join one (see below). |
| **Training Grounds** | Free play with infinite boost and no clock. Press `R` to put the ball back on the centre spot. |
| **Garage** | Choose a body (Compact, SUV, Hypercar), wheels (Round, Square, Triangle) and paint (blue, orange or match your team). Cosmetic only. |

---

## Multiplayer

Olympico uses **WebRTC data channels** (via [PeerJS](https://peerjs.com/)) to connect players directly to each other. There is no game server — the **host's browser runs the match**.

### Hosting

1. Open **Direct Connect**, enter a username and click **Host a room**.
2. Share the six-character **room code** shown at the top of the lobby.
3. Pick a team format (1v1–4v4), decide whether empty seats should be filled with bots, and click **Launch match** once everyone is in.

### Joining

- **From the session browser** — rooms hosted by other people on your network appear automatically under *Sessions on your network*, showing who is hosting, how many players are in and whether the match is already running. Click **Join**.
- **By code** — type the room code and press **Join**.

Players choose **Blue**, **Orange** or **Spectate** in the lobby. If a team is full you are placed on the other side (or as a spectator if both are full). You can also join a match that is already in progress; if bots are enabled you take over a bot's seat. When the match ends everyone is returned to the lobby so the host can launch a rematch without reconnecting.

> **Three-player rule:** with exactly three people in the lobby, bot fill is locked on so the teams stay balanced.

### Requirements and caveats

- Multiplayer must be served over `http://` or `https://` (not opened from `file://`).
- Signalling uses the public PeerJS cloud server, and the session browser uses public MQTT brokers plus a public-IP lookup to work out which rooms are "on your network". All players therefore need internet access, even on a LAN. If either service is unreachable the game says so and you can still join by room code.
- The host should be the player with the best machine and connection — their browser is the referee.

---

## How it works

Everything lives in one file, `index.html`: the CSS, the markup for the menus/HUD, and a single ES module.

### Rendering and physics

- **[Three.js](https://threejs.org/)** draws the stadium, cars, ball, boost pads and particle effects with shadows and ACES tone mapping.
- **[cannon-es](https://pmndrs.github.io/cannon-es/)** provides the rigid-body simulation, stepped at a fixed 60 Hz inside a frame-rate-independent accumulator loop. Cars are single box bodies; the arena is a set of static boxes and a ground plane; the ball is a sphere with continuous collision detection so fast shots never tunnel through walls.
- Car handling is arcade-style rather than a tyre model: while grounded, the code shapes the body's velocity directly (throttle curve, lateral grip that loosens under handbrake, speed-scaled steering rate) and applies a strong alignment torque toward the surface normal, which is what makes wall riding possible. Airborne, the same inputs become pitch/yaw/roll torques. Ground contact is detected from the arena's known geometry plus a short downward raycast.
- The **camera** is computed from the car's yaw only, then given a fixed world `up`, so it never inherits the car's pitch or roll — hence "horizon-locked".
- Bots run a small state machine (*pursue* → *challenge* → *defend*) that steers toward a point behind the ball on the line to the opponent's goal, powerslides through sharp turns, boosts on long straights, and jumps at high balls.

### Netcode

The host is authoritative for the clock, score, ball, bots, boost meters and pad state, and broadcasts a **snapshot** 30 times per second. Guests:

- simulate **their own car locally** for zero-latency control and send inputs plus their car's state 60 times per second. On the host each guest is a full physics body that is driven by those inputs and gently reconciled toward the reported state, so collisions with the ball and other cars happen in the authoritative world;
- correct their own car toward the host's view only outside a small dead-zone (or when a **reset sequence number** changes after a kickoff, in which case they snap and acknowledge — the host ignores stale guest state until it sees the ack);
- simulate a **local copy of the ball** so hits feel immediate, then blend it toward the host's ball every snapshot. A contact-based *ball-hit* hint is also sent, which the host applies only if its own physics missed the contact;
- render other players and bots as lightweight **ghosts** that are interpolated and velocity-extrapolated every frame for smooth motion between snapshots.

### Session discovery

Hosts publish a small heartbeat every few seconds to a public MQTT broker on the topic `olympico/v1/<sha256(public IP)>/<room code>`; browsers on the same public IP subscribe to that prefix and list what they hear, expiring rooms that go quiet. Leaving a room publishes a "closed" message so it disappears immediately.

### Dependencies (all CDN)

| Library | Purpose |
| --- | --- |
| `three@0.180` | 3D rendering |
| `cannon-es@0.20` | Rigid-body physics |
| `peerjs@1.5` | WebRTC signalling and data channels |
| `mqtt@5.15` | Session browser pub/sub |

---

## Development

There is no toolchain. Edit `index.html`, refresh the browser.

- Quick syntax check: extract the `<script type="module">` body to a `.mjs` file and run `node --check` on it.
- The page exposes `window.olympico` (`game`, `network`, `discovery`, `car`, `ballBody`, `input`, `remoteCars`, `ghostCars`, `botControllers`, `startGame`) for poking at the running game from the console or from automated harnesses. Two iframes of the page on one origin can host and join each other for end-to-end multiplayer tests.
- See `AGENTS.md` for more notes on the architecture and verification workflow.

## License

Released under the [MIT License](LICENSE).
