# 🍽️ MatchMeal

**Decide where to eat, together.** Share one link, everyone adds their taste on their own device, and MatchMeal finds the restaurant the whole table is happy with — then locks it in when everyone agrees.

It's a **single static web app**: no backend, no accounts, no install, no tracking. Built by [woodyhoko](https://github.com/woodyhoko).

- **Live:** https://space.hoko.xyz/match-meal/
- **Classic version:** https://space.hoko.xyz/match-meal/classic.html

---

## Table of contents

- [Two versions](#two-versions)
- [What it does](#what-it-does)
- [The four response modes](#the-four-response-modes-how-you-react)
- [How it works (architecture)](#how-it-works-architecture)
- [Restaurant data, ratings & reviews](#restaurant-data-ratings--reviews)
- [Privacy](#privacy)
- [Run it locally](#run-it-locally)
- [Deploy to GitHub Pages](#deploy-to-github-pages)
- [Project layout](#project-layout)
- [Tech stack](#tech-stack)
- [Monetization directions (notes)](#monetization-directions-notes)
- [Roadmap / not yet built](#roadmap--not-yet-built)
- [License](#license)

---

## Two versions

| | **New design — `index.html`** (default) | **Classic — `classic.html`** |
|---|---|---|
| Feel | Minimalist, mobile-first, share-first | Three-column "control room" dashboard |
| AI | Optional: on-device **or** cloud **or** none | On-device only (Chrome Prompt API → Gemma) |
| Group model | Everyone reacts; ranked by the group; all-party lock-in | Organiser proposes; agents reply on each owner's behalf |
| Best for | Phones, quick decisions, going viral | Power users, the original concierge experience |

The two are cross-linked (a "classic version" link in the new app; a "✨ New design" link in the classic one) and both ship from the same repo, so a single `git push` redeploys both.

The rest of this README focuses on the **new design** unless noted.

---

## What it does

- **Connect peer-to-peer.** One person taps **Invite** to share a link (or QR code, or the native share sheet). Whoever opens it joins instantly over a direct **WebRTC** data channel — no app, no sign-up. Three or more people automatically form a full mesh.
- **A private taste profile.** Each person taps the cuisines they **love** / **avoid**, picks dietary needs, a budget and a max distance. It's cached on their own device and shared with the table so the search fits *everyone*.
- **Find real places.** Using your location, the app searches actual nearby restaurants (OpenStreetMap) and proposes a batch, ranked by the **whole table's** combined taste.
- **Everyone reacts.** Each option gets a 😋 / 🤔 / 🙅 from every member — automatically (from taste or AI) or by tapping. Options are **ranked by the group's reactions**, best on top, with a smooth re-order animation.
- **Lock it in, together.** A place can only be locked in once **every member is in** — it's a true all-party decision, not one person forcing it. When that happens: confetti 🎉 and one-tap directions.
- **Nobody gets left "waiting."** New joiners receive a full sync of the current options *and* everyone's existing reactions, and immediately respond, so the board is never stuck.
- **Caches everything.** Your profile, settings and location persist between visits — nothing to re-enter.
- **Beautiful on phone and desktop.** Mobile is a single-column app with bottom sheets; desktop is a roomier two-column layout with a sticky control rail, a 2-up option grid, and centered modal dialogs.

---

## The four response modes (how you react)

This is the part that's easy to miss, so it's worth stating plainly. Each person chooses **one** mode (during onboarding, changeable anytime in **Settings**). It controls **how your reactions to options happen** — it does **not** affect the restaurant search itself (the search is always location + everyone's taste, no AI).

| Mode | What happens | Needs | Privacy |
|---|---|---|---|
| 👆 **I'll tap myself** | Nothing is automatic — you tap 😋/🤔/🙅 on each option. | nothing | fully private |
| ⚡ **Auto from my taste** | The app reacts instantly using your preferences (loved/avoided cuisines, distance). | nothing | fully private |
| 🧠 **Auto with on-device AI** | A language model **on your device** reads each option and reacts with nuance. | Chrome/Edge with built-in AI | fully private (stays on device) |
| ☁️ **Auto with cloud AI** | A free online model reacts for you — works on any phone. | internet | sends your picks to the online service |

Notes:
- **On-device AI** uses Chrome's built-in Prompt API (`window.LanguageModel`) — zero download, nothing leaves the device. It's shown disabled with a "needs Chrome AI" note where unavailable.
- **Cloud AI** lazy-loads a keyless provider ([Puter.js](https://developer.puter.com/)) only when you choose it, so the default experience stays dependency-free. It's hardened with timeouts that fall back to the deterministic taste reaction, so it never hangs.
- Every AI call has a deterministic fallback, so the app stays fully usable even when no model is available.

---

## How it works (architecture)

### Peer-to-peer transport
MatchMeal uses **PeerJS** purely as a tiny cloud signalling broker — it carries only the WebRTC handshake. Once two browsers connect, they talk directly over an `RTCDataChannel`; the broker is out of the loop.

- **Host vs guest.** The person who shares the link stays reachable under a short id (`tbl-XXXXXX`); friends join via `?t=<id>` (the invite link), which auto-connects on load.
- **Gossip into a full mesh.** A new peer announces itself and its known peers; everyone dials anyone they're missing, so 3+ people converge into a full mesh.
- **Flood + de-dup.** Messages are JSON envelopes flooded with a TTL and de-duplicated by id, so they reach the whole mesh even across indirect links.
- **Message types:** `HELLO` (introduce + share prefs + gossip peers), `SYNC` (full board state to a newcomer), `PROPOSE` (put options on the table), `REACT` (a reaction), `FINALIZE` (lock in the agreed place).

### Considering the whole table
Whoever searches aggregates **everyone's** shared preferences: loved cuisines (weighted by how many want them), avoided cuisines and diets (hard-excluded), the **minimum** of everyone's max distances (so all can reach), and budget. Real candidates come from OpenStreetMap; they're scored against that aggregate plus distance, and the top ~6 are proposed.

### Group ranking & all-party consensus
Each member's reaction (`in` / `ok` / `no`) is a vote. Options are sorted by the **group score** (`in` = +2, `ok` = +1, `no` = −3), best first. A place becomes lockable **only when every live member is `in`** — then any member can seal it, and everyone sees the celebration. That's the "decided by all parties" rule.

### On-device intelligence (optional)
See [the four modes](#the-four-response-modes-how-you-react). The "AI" in this app is used **only to react to options on your behalf** — searching and ranking are deterministic and need no AI. Reasoning steps request a strict JSON shape (used like a function call) and are parsed with a brace-balanced extractor that survives chatty models.

---

## Restaurant data, ratings & reviews

- **Places** come from the **OpenStreetMap Overpass API** (keyless, with mirror fallbacks). Distances are computed locally with the haversine formula. **Nominatim** handles typing in a location by name.
- **Ratings on each card:** every option shows a **"Table match"** rating — a deterministic score (★/5) of how well the place fits the *whole table's* taste (loved cuisines, distance), computed on-device. When anyone is in an **AI mode**, the card also shows an **AI rating** (the average of the agents' 1–5 scores, produced via the model's structured/function-call output) and that agent's one-line take. So you get a useful rating even with no AI, and a smarter one when AI is on.
- **Real reviews & photos:** every option links straight to its **Google Maps** page (the "★ reviews & photos" link), and the locked-in place's **Directions** open in Google Maps — Google's real ratings, reviews and photos in one tap, free, no API key.
- **Why not embed Google's reviews inline?** That requires the paid **Google Places API** with a billable key — unsafe to embed in a public static page and restricted by Google's terms. OpenStreetMap rarely carries ratings or photos, so Google (via the link) is the right source for those. Embedding them is on the [roadmap](#roadmap--not-yet-built) as an optional, key-required feature.

---

## Privacy

- Your raw tastes, rules and location are only ever fed to **your own** device (and to your own local model in on-device AI mode). They are stored in `localStorage`, not on any server.
- The **only** things sent over the WebRTC channel are the reactions and answers you choose to share — never your full profile's private internals.
- The one exception is **Cloud AI mode**, which (by your explicit choice) sends the option + your taste to the online provider to get a reaction. It's clearly labelled and entirely opt-in.
- Because it's a purely client-side app served as static files, anyone can View-Source it — the code is protected by the [LICENSE](#license), not by secrecy.

---

## Run it locally

Serve over `http(s)` — geolocation, WebGPU and OPFS are blocked on `file://`, so opening the file directly won't work for location or AI.

```bash
python -m http.server 8000
```

Then open **http://localhost:8000** in **Chrome or Edge**. (Connecting, profiles and the deterministic "taste" mode work in any modern browser; on-device AI needs Chromium with the built-in Prompt API.)

---

## Deploy to GitHub Pages

The repo is already wired for this. Any push to `main` runs `.github/workflows/pages.yml`, which **auto-enables and deploys** Pages — no Settings clicks needed. The site is served at `https://woodyhoko.github.io/match-meal/` and (via custom domain) `https://space.hoko.xyz/match-meal/`.

To set it up from scratch on another account:

1. Create a **public** repo (Pages on a free plan needs public).
2. Push the files:
   ```bash
   git init
   git add .
   git commit -m "MatchMeal"
   git branch -M main
   git remote add origin https://github.com/<you>/match-meal.git
   git push -u origin main
   ```
3. The included workflow turns Pages on automatically. (Or, without the workflow: **Settings → Pages → Deploy from a branch → `main` / `/root`**.)

A social/link-preview image (`og.jpg`) and OpenGraph/Twitter meta tags are included, so shared links unfurl with a proper card. After first sharing, prime the cache via the [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/) / X Card Validator.

> **Note:** GitHub Pages serves the client-side source as-is — that's true of every front-end app. Protection is the LICENSE, not secrecy.

---

## Project layout

```
index.html                 The app — new design (v2). Single self-contained file.
classic.html               The original version (v1). Single self-contained file.
og.jpg                     1200×630 social / link-preview image.
.github/workflows/pages.yml  Auto-enable + deploy to GitHub Pages on push.
LICENSE                    Proprietary, all-rights-reserved.
README.md                  This file.
```

Each `*.html` is fully self-contained (HTML + CSS + vanilla JS, no build step). Dependencies load from HTTPS CDNs at runtime.

---

## Tech stack

Vanilla **HTML / CSS / JS** · **PeerJS** over WebRTC data channels · **OpenStreetMap** Overpass + Nominatim · **Google Maps** deep links (reviews/photos/directions) · Chrome built-in **Prompt API** (on-device AI) · **Puter.js** (optional keyless cloud AI) · **QRCode.js** (invite QR). The classic version additionally uses **MediaPipe LiteRT GenAI** + **Gemma** (`gemma-4-E2B-it`) for its on-device fallback. No build, no backend, no framework. The peer-to-peer + on-device-AI approach builds on the author's earlier [ai-mesh](https://github.com/woodyhoko/ai-mesh) project.

---

## Monetization directions (notes)

MatchMeal is free and viral by design; the realistic strategy keeps the share loop free forever and monetizes asymmetrically. Directions being explored, roughly in order:

1. **Validate the high-intent moment** — a small "Get directions / Order delivery / Book a table" action row that appears **after** a place is locked in (the highest-intent instant in dining). Instrument: how many sessions reach lock-in, and how many tap the row.
2. **Manufacture frequency** — target **work-lunch crews** (the one recurring, co-located use case): saved crews, one-tap "lunch again", a "we're at X" share-out. Optimise *returning tables*, not installs.
3. **Turn on affiliate revenue** (delivery/reservation CPA) on that action row once volume exists — strictly downstream of the free choice, never biasing the ranking.
4. **B2B is the real ceiling** — license the privacy-preserving group-decision engine as an embeddable widget (HR/perks, travel, events, dating, team tools), where "preferences never leave the device" is a paid compliance/integration feature; plus low-effort commercial self-host / de-branding licenses.

**The one rule:** never put a paywall or login in the join flow — the viral share is the only acquisition channel.

---

## Roadmap / not yet built

- **Tinder-style swipe deck (mobile)** — swipe to react, hide a card once you've decided, with a dedicated "match board" view. Pending a usable photo source (see below) and a clean way to reconcile swiping with the auto-respond modes.
- **Restaurant photos** — there is no reliable *free, keyless* source of real location-specific restaurant photos (Google Places needs a paid key; OpenStreetMap rarely has images; Wikimedia/Wikidata cover only a few notable venues). Likely path: optional, behind a user-supplied key, or AI-generated cuisine art as a decorative placeholder.
- **Optional inline Google ratings** — show real Google ratings/reviews on the card itself, behind a user-supplied, referrer-restricted Places API key.
- **`eat.hoko.xyz`** — a dedicated subdomain for the app.
- **Simplify / unify** onboarding and settings copy across both versions.

### Recently added

A single **consolidated rating** per card (AI score when an AI mode is on, otherwise the deterministic table-match), a **match board** banner showing the current front-runner, **visible rejection reasons** (who turned a place down and why), a **custom free-text preference**, **per-restaurant decision history** that auto-applies when you rejoin a table and see the same place, **name/avatar editing**, a **"who's at the table"** view, and **clear-history**.

Plus a **rendering pass for smoothness**: card updates are now **region-level and flash-free** (reacting on one card — or a friend's reaction arriving — only touches the bits that changed, never rebuilds other cards, and reaction buttons just toggle state rather than re-render), an **"AI thinking…"** indicator with a gentle fade-in for the AI's comment, a **stable card order** while everyone responds (with an opt-in **"Sort by best match"** button — the leader is always badged 🥇 even when it's not at the top), and **equal-height cards per row** on desktop. Card internals are aligned so the consensus bar's position doesn't depend on how long the AI's comment is.

And some session/quality-of-life items: the lock-in shows **who locked it in** (card + celebration), the celebration opens **Google Maps** for directions, the **invite link is stable across refreshes** and you **auto-rejoin your last table** when you reload, and there's a **Saved choices** manager to review, re-rate, or remove your remembered decisions. Ranking breaks ties on the **rating/grade** (so among equally-liked options the better-matched one wins), switching **into an AI mode re-assesses every option** on the table with fresh AI comments, and the free-text "Anything else?" note is labelled **AI-only** (it's read only in an AI mode).

Latest polish: others' comments are **private until you tap their face** (hover shows their status, e.g. "Ada is deciding…"); reacting **no longer nudges other cards** (re-rank animation runs only when the order truly changes, and the leading banner keeps a fixed height); the **Saved choices** list now **sorts** by recent / name / rating / visits / nearby, tracks a per-place **visit count**, lets you **open it in Maps** and **add your own note**, and never re-sorts mid-edit; the top-right **status pill opens "who's at the table"**; and a new **Past tables** view shows the tables you've been in and **what got locked in**, with one-tap rejoin.

---

## License

This software is **proprietary**.

**Copyright © 2026 woodyhoko. All Rights Reserved.**

Reuse, copying, modification, redistribution or hosting of this code is **not permitted**. Viewing the publicly served source does not grant any license to it. See the [LICENSE](LICENSE) file for the full terms.
