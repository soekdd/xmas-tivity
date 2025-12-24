# Axtivity – Christmas Activity Deck

Axtivity is a Vue 3 + Vite + Vuetify single-page app that acts as a card deck for the tabletop game Activity. It serves Christmas-themed tasks across the classic categories (drawing, performance, talk) and difficulty levels 3–5. The app runs fullscreen as a PWA and prevents cards from repeating during a session.

## Features
- Task pool by difficulty (3–5) and category (drawing, performance, talk) with Christmas wording.
- Guided flow: pick a level and category, see a card for 10s, play for 60s, skip if needed, confirm the result, then reveal the word.
- Infrequent “public” rounds (roughly 1 in 6) that need confirmation before showing.
- PWA-ready: manifest, service worker, and icons for homescreen installation and fullscreen start.
- Swappable assets (backgrounds, card art) in `src/assets/`.

## Getting started
Requirements: Node.js 18+ and npm.

```bash
npm install
npm run dev
```
Open `http://localhost:5173` in your browser.

## Build and preview
Create a production bundle:
```bash
npm run build
```
Serve the built app locally:
```bash
npm run preview
```
The output lives in `dist/` with relative paths, so it can be hosted from a subfolder.

## Deploy
Use the helper script from the project root:
```bash
./deploy.sh
```
It syncs `dist/` to `soek@soek.de:/var/www/soek.de/xmas/` via `rsync`. Edit `deploy.sh` if you deploy elsewhere.

## Project layout
- `src/App.vue` – main UI and gameplay flow.
- `src/data/tasks.js` – Christmas task lists grouped by level and category.
- `src/plugins/vuetify.js` – Vuetify setup for theming and components.
- `src/style.css` – global styles and background.
- `public/` – manifest, icons, service worker, and audio assets.
- `vite.config.js` – Vite config with Vuetify plugin and relative base.

## Customization
- Add or edit tasks in `src/data/tasks.js`.
- Swap card art or backgrounds in `src/assets/`.
- Adjust timers or flow in `src/App.vue`.
