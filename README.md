play at https://fercasti.github.io/real-or-ai/
# Real or AI — **AI Race**

A browser **endless runner** built with **Three.js** (WebGL). You drive down a split highway: one side is labeled **AI**, the other **REAL**. The run is about reacting fast—stay in the lane that matches the moment, dodge obstacles, collect coins for score, and use a short speed boost when you need it.

<p align="center">
  <img src="REALvsAI.png" alt="Real or AI — AI Race" width="520" />
</p>

## Play online

- **[https://fercasti.github.io/real-or-ai/](https://fercasti.github.io/real-or-ai/)**

## What this repo contains

| Area | What it is |
|------|----------------|
| **`projects/game/public/`** | The full game: single-page **`index.html`** with embedded UI, Three.js scene, audio hooks, and procedural/neon visuals. This is what gets hosted. |
| **`projects/game/public/assets/`** | In-game assets (models, textures, audio, etc.) referenced by the game. |
| **`projects/game/vercel.json`** | Optional static-hosting config if you deploy the `public` folder elsewhere (e.g. Vercel). |
| **Root `.glb` / images** | Extra 3D and image assets used for experimentation or references outside the main `public` bundle. |
| **`Universal Base Characters[Standard]/`** | Character model pack (glTF) included in the repo for reference or future use. |

The shipped experience is **one run at a time**: wrong lane choice or a crash ends the run; the UI covers start, pause, resume, retry, and score/coin feedback.

## Controls

- **A** — move toward the **AI** side (left lanes).
- **D** — move toward the **REAL** side (right lanes).
- **W** — **speed boost** (limited use; shown in the HUD when active).

Touch and on-screen hints mirror the same idea (AI vs REAL lanes).

## Run locally

From the repo root, serve the **`projects/game/public`** folder over HTTP (so asset paths resolve correctly), then open the site root in a browser.

Example using a static server:

```bash
npx --yes serve "projects/game/public"
```

Then open the URL the tool prints (usually something like `http://localhost:3000/`).

## Tech notes

- **Three.js** `WebGLRenderer` on a fixed **16:9** play area, HUD overlays, lane-based movement, obstacles, coin pickups, timer/score display, and lightweight audio.
- **No build step** for the live game: it is static files under `projects/game/public/`.

## Repository layout (short)

```text
projects/game/public/   ← play this (index.html + assets)
projects/game/          ← game-level config (e.g. vercel.json)
README.md               ← you are here
```

If you fork or republish, keep the same origin for `public/` so relative asset URLs keep working.
