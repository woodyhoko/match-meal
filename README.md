# MatchMeal

**A peer-to-peer "where shall we eat?" negotiator — your tastes stay on your device, your AI does the haggling.**

MatchMeal connects a small group of friends over a direct browser-to-browser link and helps them agree on one restaurant everyone is happy with. Each person keeps their own dining preferences entirely on their own device, where a private on-device AI "concierge" searches the real world for nearby places, proposes options, and — on its owner's behalf — answers everyone else's agents about whether a suggestion works. The organiser keeps proposing until every agent at the table is satisfied, then locks in the pick. Nothing about your tastes ever leaves your machine; only the verdicts your agent chooses to share are sent over the wire.

It is a single `index.html` file: no build step, no backend, no framework, no accounts.

## Features

- **Direct peer-to-peer connection.** Friends join over a real WebRTC data channel via an invite link; three or more people automatically form a full mesh.
- **A private taste profile that lives on your device.** Cuisines marked love/avoid, dietary needs, a max budget, a max distance and free-text notes — all cached locally so you never re-enter them.
- **A personal AI concierge.** An on-device model plans the search, picks the best restaurant, writes a short pitch, and represents you in the negotiation — all without any of your data leaving the browser.
- **Real OpenStreetMap restaurant search from your location.** Keyless, CORS-enabled queries find genuine nearby places using your geolocation, with distances computed locally.
- **Agents that negotiate on each owner's behalf.** Every participant's agent judges a proposal against *its own* owner's private preferences and replies satisfied / concern / veto, with a reason.
- **Learning from a rejection.** When you turn an option down, the app offers to remember it as a rule — never suggest that place again, allow it only under a condition, or avoid that cuisine entirely.
- **Everything is cached.** Profile, rules, location and the (large, one-time) model weights are all persisted, so repeat visits are instant and offline-friendly.
- **One-click import / export.** Save your preferences to a portable `*.matchmeal.json` file and load them back by file picker or paste.
- **Responsive UI for desktop and mobile.** A warm, food-themed glassmorphism design that flows from a three-column desktop layout to a single mobile column.

## How it works (the approach)

### Peer-to-peer transport

MatchMeal uses **PeerJS** purely as a tiny cloud signalling broker — it carries only the WebRTC SDP/ICE handshake. The moment two browsers have exchanged that handshake, they talk directly to each other over an `RTCDataChannel`; the broker is no longer in the conversation.

- **Host vs guest.** Anyone who keeps "Stay reachable so others can join me (host)" enabled stays registered on the broker under a short id (e.g. `meal-AB12CD`) and is auto-reconnected if the broker connection drops. Friends dial in either by pasting that id or by opening an invite link of the form `…/?join=<id>`, which auto-connects on load.
- **Gossip into a full mesh.** When a peer connects it sends a `HELLO` carrying its name, emoji and *its own list of known peers*. Each side then dials any peers it doesn't already have, so a group of three or more converges into a full mesh where everyone is directly connected (connections learned this way are marked "relayed" in the UI).
- **Flood with TTL + de-dup.** Every message is a JSON **envelope** — `{ v, id, type, from, fromName, fromEmoji, ttl, ts, payload }`. Non-`HELLO` events are handled locally and then re-broadcast to every other connection with `ttl - 1`, so messages reach the whole mesh even across indirect links. A bounded `seen` set keyed by envelope `id` discards duplicates, so flooding never loops.
- **Message types:** `HELLO` (introduce self + peer list), `PROPOSE` (put a restaurant on the table), `VERDICT` (an agent's stance on a proposal), `ASK` (ask the table a question), `ANSWER` (an agent's reply on its owner's behalf), `FINALIZE` (lock in the agreed place) and `CHAT` (a plain human message to everyone).

### On-device intelligence

Each user runs their **own** language model locally — there is no shared server model.

- **Chrome built-in Prompt API preferred.** If `window.LanguageModel` is available, the concierge reasons through it with zero download.
- **On-device Gemma fallback.** Otherwise it falls back to Google **Gemma** (`gemma-4-E2B-it`) running locally via **MediaPipe LiteRT / WebGPU**. The MediaPipe GenAI runtime is **dynamically imported only when Gemma is actually needed**, so Prompt API users never fetch it and the page itself loads instantly.
- **Weights cached in OPFS.** The first Gemma run is a one-time (~2 GB) download streamed with a progress bar and cached in the browser's Origin Private File System (with a stored size sentinel for validation), so subsequent visits skip the download entirely.
- **Schema-constrained JSON as "function-calling."** The concierge's reasoning steps (plan a search, pick the best candidate, cast a verdict, answer the table) each request a strict JSON shape — used like function calls — and the response is parsed with a balanced-brace extractor that survives chatty models.
- **Deterministic fallbacks everywhere.** Every AI call (`AI.json`) ships with a hand-written fallback result, so if no model is present, the model errors, or the JSON is malformed, the app still produces a sensible answer and stays fully usable. Hard preference rules (never-suggest a place, avoid a cuisine) are also enforced *deterministically before the model is ever consulted*, so private vetoes never depend on the LLM.
- **Privacy.** Your raw tastes, rules and notes are only ever fed to your own local model. The only things that travel over the data channel are the verdicts and answers your agent chooses to share.

### The negotiation protocol

1. **Propose.** Your concierge calls the search tool, filters and scores the results against your loves/avoids and rules, picks one, writes a one-sentence pitch, and broadcasts a `PROPOSE`.
2. **Verdict.** On receiving a proposal, *each* participant's agent evaluates it against its **own** owner's private profile and replies with a `VERDICT` of **satisfied**, **concern** or **veto**, plus a short first-person reason. The proposer's own agent records an honest verdict too.
3. **Finalise.** The card shows every participant's stance (with reasons on hover). When everyone is satisfied and no one has vetoed, the proposer can `FINALIZE` — and any participant can manually override their own agent's stance, because the human always has the final say.
4. **Ask the table.** The organiser can poll everyone's mood with an `ASK`; each agent answers (`ANSWER`) what its owner is in the mood for, and those moods feed the next search plan.
5. **Remember a rejection.** When you mark a proposal as a concern or veto, MatchMeal offers to store that dissatisfaction as a reusable rule — never again, only-under-a-condition, or avoid that cuisine.

**One round at a glance:**

```
your concierge
   │  search_restaurants(near, cuisines, radius)   ← OpenStreetMap Overpass
   │  pick best + write pitch
   ▼
 PROPOSE ──flood──▶ every peer
                      │  each agent evaluates vs. its OWN owner's profile
                      ▼
                   VERDICT (satisfied / concern / veto + reason) ──flood──▶ all
   ┌───────────────────────────────────────────────┐
   │ all verdicts in & nobody vetoed?               │
   └───────────────────────────────────────────────┘
                      │ yes
                      ▼
                  FINALIZE ──flood──▶ "🎉 it's settled"
```

### Real-world data

The "tool" the concierge calls is real-world search, all keyless and CORS-friendly:

- **OpenStreetMap Overpass API** finds real nearby restaurants, cafés and fast-food places within your chosen radius from your geolocation. Multiple mirror endpoints are tried in turn for resilience, and dietary tags (vegetarian / vegan / halal) are read straight from OSM data.
- **OpenStreetMap Nominatim** handles forward geocoding (look up a place name you type) and reverse geocoding (turn your coordinates into a neighbourhood label).
- **Haversine formula** computes the great-circle distance to each candidate locally, used for both display and ranking.

## Run it locally

MatchMeal **must be served over http(s)** — geolocation, WebGPU and OPFS are all blocked on `file://`, so opening `index.html` directly will not work for the AI or location features.

From the project folder:

```bash
python -m http.server 8000
```

Then open **http://localhost:8000** in **Chrome or Edge**.

> The AI features need a Chromium-based browser: either one with the built-in Prompt API, or one that supports WebGPU for the one-time on-device Gemma download (~2 GB, cached afterward). Other browsers can still connect, set a profile and chat, but the concierge reasoning needs a supported browser.

## Deploy to GitHub Pages

Because the app is a single static file with no backend, GitHub Pages hosts it perfectly. These steps assume the GitHub user **woodyhoko** and a repo named **match-meal**, served from the root of the `main` branch.

1. **Create the repository** `match-meal` under your account at https://github.com/new. A **public** repo is the simplest setup and is assumed below.
2. **Push the code:**
   ```bash
   git init
   git add index.html LICENSE README.md
   git commit -m "MatchMeal: peer-to-peer dining negotiator"
   git branch -M main
   git remote add origin https://github.com/woodyhoko/match-meal.git
   git push -u origin main
   ```
3. **Enable Pages:** in the repo, go to **Settings → Pages → Build and deployment → Source: Deploy from a branch**, then choose **Branch: `main`** and **Folder: `/ (root)`** and click **Save**.
4. After a minute, your site is live at:

   **https://woodyhoko.github.io/match-meal/**

   GitHub Pages is served over **HTTPS**, which is exactly what geolocation and WebGPU require — so the full app works on the deployed URL.

> **Important — Pages serves your client-side source as-is.** This is a purely client-side app, so anyone who visits the site can read its source. That is unavoidable for any front-end web app, and a public repo (the simplest way to host on Pages) makes that source openly browsable too. The code is **protected by the LICENSE (copyright), not by secrecy** — viewing the source does not grant any right to copy, reuse or redistribute it.

## Tech

Vanilla HTML / CSS / JS · PeerJS over WebRTC data channels · OpenStreetMap Overpass + Nominatim · MediaPipe LiteRT GenAI · Google Gemma (`gemma-4-E2B-it`) · Chrome built-in Prompt API — no build, no backend, one file. All dependencies load from HTTPS CDNs; the only same-origin reference is `index.html` itself, so it runs unchanged at any base path. The peer-to-peer + on-device-AI architecture builds on the author's earlier [ai-mesh](https://github.com/woodyhoko/ai-mesh) project.

## License

This software is **proprietary**.

**Copyright (c) 2026 woodyhoko. All Rights Reserved.**

Reuse, copying, modification, redistribution or hosting of this code is **not permitted**. Viewing the publicly served source does not grant any license to it. See the [LICENSE](LICENSE) file for the full terms.
