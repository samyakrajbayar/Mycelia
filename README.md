# Mycelia

> A single-page generative web app that renders a living, breathing mycelial network. Click anywhere to plant a spore — a glowing thread germinates and grows. Built for **CodeStorm 2026 #2 — "Create Websites That Feel Alive."**

This README documents what's actually implemented in `index.html` right now, what's stubbed out for later, and how to wire up the missing pieces (multiplayer + the AI Oracle).

---

## What's working today

Open `index.html` in any browser — no build step, no dependencies, no server required.

- **Canvas-based growth engine.** A branching algorithm seeds 4 root nodes on load and grows organic, recursive threads outward (`spawnBranch`), giving the "establishing growth" hero moment described in the brief.
- **Click/tap to plant.** Clicking the canvas drops a new spore and grows a multi-branch thread from it in real time.
- **Keyboard accessibility.** Press `P` to plant a spore at a random position — satisfies the "focusable, non-mouse" interaction requirement.
- **Ambient motes.** Particles continuously spawn and drift from outer nodes back toward the roots, independent of user action, as the "this is alive" signal.
- **Day/night tint.** New growth is colored amber in the day and `petrichor-blue` at night, based on the visitor's local clock hour (`determineTint()`).
- **The Oracle panel.** A slide-in panel with a text input. On submit, it shows a short, cryptic, organism-voiced line and animates a light pulse traveling across the existing network to a root node, timed to arrive with the text.
- **Live telemetry readout.** Bottom-left glass panel showing node count, "tending now" count, and the current weather/time label, updating continuously.
- **`prefers-reduced-motion` support.** Growth speed and mote spawning respond to the OS-level setting.
- **Responsive layout.** Oracle panel goes full-width and telemetry/instruction text resizes under 600px.
- **Design tokens.** The full `humus` / `hypha` / `bioglow` / `spore-amber` / `petrichor-blue` / `bone-linen` palette and the Fraunces / Inter / IBM Plex Mono type system are implemented exactly as specced.

## What's stubbed, not yet live

These features have working UI and fallback behavior, but the real backend integration is commented out or simulated:

| Feature | Current behavior | What's missing |
|---|---|---|
| **Multiplayer / shared network** | Runs in **Solo Mode** — `connectionStatus` reads "SOLO MODE", and a `setInterval` fakes 1–2 presence dots so the screen never looks empty. | Firebase Realtime Database config block exists in HTML but is commented out. Uncomment it, paste a real Firebase config, and fill in `onValue` listeners + `syncNodeToDB()` (currently a no-op stub) to sync nodes across real clients. |
| **Weather** | Tint is based on local device hour only (day = amber, night = blue). | No Open-Meteo call yet. Geolocation permission is requested and acknowledged, but the fetch to `api.open-meteo.com` is not made — see the comment in the geolocation success callback for the exact endpoint shape to drop in. |
| **The Oracle's AI reply** | Always uses a local fallback generator — 8 hardcoded cryptic lines, picked at random, after a simulated 1.2–2.2s "thinking" delay. | The serverless `fetch('/api/oracle', ...)` call is written out in a comment block directly above the fallback logic. Stand up that endpoint (Vercel/Netlify function calling an LLM server-side) and swap in the real call, keeping the existing fallback as the error/timeout path so the demo never visibly breaks. |

## File structure

This is currently a single self-contained file:

```
index.html   ← everything: markup, CSS (custom properties for the palette), and JS engine
```

The JS is organized in commented sections mirroring the original day-by-day build plan:

```
MYCELIA ENGINE              → state, canvas setup, color utilities
DAY 3: ENVIRONMENTAL TINTING → determineTint(), geolocation handling
DAY 1: GROWTH ALGORITHM      → seedInitialNetwork(), plantSpore(), spawnBranch()
DAY 1: MOTES                 → spawnMote(), updateMotes()
DAY 4: ORACLE PULSES         → spawnOraclePulse(), findPath() (BFS up the tree)
RENDERING                    → drawNetwork()
DAY 2: REALTIME SYNC         → Firebase stub, syncNodeToDB()
DAY 4: THE ORACLE             → panel open/close, fallback lines, submit handler
EVENT LISTENERS & INIT
```

## Running it

No install needed:

```bash
# any static server works, e.g.
npx serve .
# or just double-click index.html
```

To test the "shared network" feel before multiplayer is wired up, open the file in two tabs — each tab will run its own independent solo simulation rather than a synced one, until Firebase is connected.

## Wiring up multiplayer (Firebase Realtime Database)

1. Create a free Firebase project and enable **Realtime Database**.
2. Uncomment the `<script type="module">` block near the top of the body and paste your config into `firebaseConfig`.
3. In `syncNodeToDB(node)`, replace the no-op with a real `set(ref(db, 'nodes/' + node.id), {...})` write (the shape is already sketched in the comment above it).
4. Add an `onValue` listener on `nodes/` that pushes any node you didn't create yourself straight into the local `nodes` array and `activeGrowth` queue, so remote growth animates in using the existing `drawNetwork()` pipeline.
5. Use `onDisconnect` + a `presence/` path to drive `presencePoints` from real visitors instead of the simulated interval.

## Wiring up the real Oracle

1. Deploy a serverless function (Vercel/Netlify) at `/api/oracle` that accepts `{ prompt }` and calls an LLM with a short system prompt: *one cryptic, organism-voiced line, ≤20 words.*
2. Keep the API key server-side only — never in client JS.
3. In the `oracleForm` submit handler, swap the `setTimeout` simulation for the real `fetch('/api/oracle', ...)` call already drafted in the comment block.
4. Leave `fallbackLines` in place as the `catch` path so a failed or slow call degrades gracefully instead of breaking the demo.

## Design reference

| Token | Hex | Use |
|---|---|---|
| `humus` | `#14110F` | Background |
| `hypha` | `#3A4A3E` | Resting thread |
| `bioglow` | `#8FFFC0` | Active growth / Oracle pulse |
| `spore-amber` | `#F2A65A` | Daytime tint / CTA |
| `petrichor-blue` | `#6FA8C7` | Night/rain tint |
| `bone-linen` | `#EDE6D6` | Foreground text |

Fraunces (display/Oracle voice) · Inter (UI) · IBM Plex Mono (telemetry).

## Known limitations

- No build tooling — everything is inline in one HTML file, which is fine at this scale but will get unwieldy if more features are added.
- Pathfinding for the Oracle pulse (`findPath`) only walks straight up the tree to a root, not a true shortest-path across the whole graph — fine visually, not technically a network path.
- No persistence between sessions in solo mode; refreshing resets the network to 4 fresh seeds.
