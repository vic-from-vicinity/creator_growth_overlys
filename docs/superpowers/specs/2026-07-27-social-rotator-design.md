# Creator Growth Overlays — Social Rotator MVP Design

**Date:** 2026-07-27
**Status:** Approved
**Scope:** Phase 1 — Backend infrastructure + Social Rotator widget end-to-end

---

## 1. Project Vision

Build standalone HTML/CSS/JS streaming overlays that help creators grow their communities. The overlays run as browser sources in Lumia Stream / OBS. A local Node.js backend serves widgets, manages configuration, and pushes live updates via WebSocket. The architecture is extensible so future widgets (YouTube promoter, goal bar, chat hype, etc.) plug into the same foundation.

---

## 2. High-Level Architecture

Three-layer local system running on the streamer's laptop:

```
┌─────────────────────────────────────────┐
│  OBS / Lumia (Browser Sources)          │
│  ┌────────────────────────────────┐    │
│  │ Social Rotator Widget (HTML)   │    │
│  │  • Renders icon + handle       │    │
│  │  • CSS animations only         │    │
│  └───────────────┬────────────────┘    │
│                  │ WebSocket            │
└──────────────────┼─────────────────────┘
                   │
┌──────────────────┼─────────────────────┐
│  Local Node.js Backend (port 3000)    │
│  ┌─────────────┐  ┌────────────────┐   │
│  │ Express HTTP│  │ WebSocket Hub  │   │
│  │ • Serve     │  │ • Push config  │   │
│  │   widgets   │  │   changes      │   │
│  │ • Serve     │  │ • Broadcast    │   │
│  │   dashboard │  │   to all       │   │
│  └─────────────┘  │   clients      │   │
│                   └────────────────┘   │
│  ┌────────────────────────────────┐     │
│  │ Config Manager                 │     │
│  │ • Read/write config.json       │     │
│  │ • Serve to widgets & dash     │     │
│  └────────────────────────────────┘     │
└────────────────────────────────────────┘
                   │ HTTP
┌──────────────────┼─────────────────────┐
│  Dashboard (localhost:3000/dashboard) │
│  • Add/remove platforms             │
│  • Set handles & rotation speed      │
│  • Pick theme preset               │
│  • Live preview                     │
└────────────────────────────────────────┘
```

**Key Principle:** Widgets are dumb and beautiful. The backend is smart. Widgets only know how to render and animate. The backend fetches data, maintains state, and pushes updates.

---

## 3. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Backend | Node.js + Express | Lightweight, fast to set up, huge ecosystem |
| Real-time | `ws` (WebSocket library) | Native WS, no Socket.IO bloat needed |
| Frontend Widgets | Vanilla HTML/CSS/JS | Lumia/OBS browser sources require no build step |
| Dashboard | Vanilla HTML/CSS/JS | Same simplicity; can upgrade to React/Vue later for the platform |
| Config Storage | `config.json` + `.env` | Simple file-based for MVP; migrate to DB when building the hosted platform |
| Styling | CSS Custom Properties (variables) | Enables dynamic theming without JS re-renders |

---

## 4. Directory Structure

```
creator_growth_overlays/
├── server.js                  # Entry point: Express + WS server
├── package.json
├── .env                       # API keys & secrets (gitignored)
├── config.json                # Widget configurations
├── .gitignore
├── README.md
├── public/
│   ├── widgets/
│   │   └── social-rotator.html      # Standalone browser source widget
│   └── dashboard/
│       └── index.html               # Config UI
├── src/
│   ├── backend/
│   │   ├── websocket.js             # WS hub logic
│   │   ├── config-manager.js        # config.json read/write
│   │   └── routes.js                # Express routes
│   └── themes/
│       └── presets.css              # Theme definitions
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-07-27-social-rotator-design.md
```

---

## 5. Backend Design

### 5.1 Express Server
- **Port:** 3000 (configurable via `PORT` env var)
- **Static files:** `public/` served at root (`/`)
- **Dashboard route:** `/dashboard` → serves `public/dashboard/index.html`
- **API routes:**
  - `GET /api/config` → returns current `config.json`
  - `POST /api/config` → updates `config.json` and broadcasts change to all WS clients
  - `GET /api/health` → simple health check

### 5.2 WebSocket Hub (`ws` library)
- **Connection:** Widgets connect to `ws://localhost:3000` on load
- **Message format:** JSON `{ type: string, payload: object, widgetId?: string }`
- **Broadcast pattern:** When config changes via dashboard, server broadcasts `{ type: 'configUpdate', payload: <fullConfig> }` to all connected clients
- **Client handling:** Maintain a `Set` of connected sockets. On disconnect, remove from set.

### 5.3 Config Manager
- **File:** `config.json`
- **Structure:**
  ```json
  {
    "socialRotator": {
      "enabled": true,
      "rotationSpeed": 5,
      "theme": "sleek",
      "platforms": [
        { "name": "Twitch", "handle": "@username", "url": "https://twitch.tv/username", "icon": "twitch.svg" },
        { "name": "YouTube", "handle": "@username", "url": "https://youtube.com/@username", "icon": "youtube.svg" },
        { "name": "TikTok", "handle": "@username", "url": "https://tiktok.com/@username", "icon": "tiktok.svg" },
        { "name": "Rumble", "handle": "@username", "url": "https://rumble.com/user/username", "icon": "rumble.svg" },
        { "name": "Kick", "handle": "@username", "url": "https://kick.com/username", "icon": "kick.svg" }
      ],
      "customPlatforms": []
    }
  }
  ```
- **Validation:** On write, validate required fields (`name`, `handle`, `url`). Reject malformed entries.
- **Defaults:** If `config.json` missing, generate with empty platform list and default theme.

---

## 6. Dashboard Design

### 6.1 Layout
Single-page app. No build step. Plain HTML/CSS/JS.

**Sections:**
1. **Header:** Project name + connection status indicator (green dot = backend reachable)
2. **Platform Manager:**
   - List of current platforms (built-in + custom)
   - Each row: Icon preview | Name | Handle | URL | Remove button
   - "Add Custom Platform" button → form with Name, Handle, URL, Icon URL
3. **Settings:**
   - Rotation speed slider (1–30 seconds)
   - Theme dropdown (Sleek, Bold, Retro, Animated)
4. **Live Preview:**
   - Embedded iframe pointing to `http://localhost:3000/widgets/social-rotator.html`
   - Updates in real-time as config changes

### 6.2 Interaction Flow
1. User modifies config in dashboard
2. Dashboard sends `POST /api/config` with updated JSON
3. Backend writes to `config.json`
4. Backend broadcasts `configUpdate` to all WS clients (including the preview iframe)
5. Widget receives message, updates CSS variables and platform list, continues rotating

---

## 7. Social Rotator Widget Design

### 7.1 Visual Spec
- **Dimensions:** 400px × 120px (flexible via OBS scaling)
- **Background:** Transparent (for OBS compositing)
- **Content:** Platform icon (48×48px) + platform name + handle text
- **Typography:** Sans-serif, platform name bold, handle regular/gray
- **Layout:** Icon left, text right, vertically centered

### 7.2 Animation
- **Transition:** Crossfade (opacity 0→1). Slide-in animation can be added as a theme variant in Phase 2.
- **Duration:** 0.5s CSS transition
- **Rotation:** JavaScript `setInterval` driven by `rotationSpeed` config value
- **No external animation libraries:** Pure CSS transitions + JS class toggling. Keeps widget lightweight.

### 7.3 Data Flow
1. Widget loads, reads `window.location.search` for optional override params
2. Opens WebSocket connection to `ws://localhost:3000`
3. On `configUpdate` message:
   - Update `platforms` array
   - Update `rotationSpeed`
   - Apply theme by setting `document.body.setAttribute('data-theme', themeName)`
4. Begin rotation loop

### 7.4 Theme System
CSS custom properties drive all visual styling:
```css
:root {
  --bg-color: transparent;
  --text-color: #ffffff;
  --accent-color: #9146ff;
  --font-family: 'Inter', sans-serif;
  --icon-size: 48px;
  --transition-speed: 0.5s;
}

[data-theme="sleek"] { /* minimal, dark, subtle glow */ }
[data-theme="bold"] { /* high contrast, neon borders */ }
[data-theme="retro"] { /* pixel font, scanlines */ }
[data-theme="animated"] { /* pulsing, particle-ready */ }
```

Widget applies theme by setting `data-theme` attribute on body.

---

## 8. Error Handling

| Scenario | Behavior |
|----------|----------|
| Backend offline | Widget shows fallback message "Waiting for backend..." with retry loop every 3s |
| Config file corrupted | Backend regenerates with defaults, logs warning |
| Invalid platform URL | Dashboard shows inline error, prevents save |
| WebSocket disconnect | Widget attempts reconnect with exponential backoff (1s, 2s, 4s, max 30s) |
| No platforms configured | Widget shows "Add your socials in the dashboard" placeholder |

---

## 9. Lumia / OBS Integration

- **Browser source URL:** `http://localhost:3000/widgets/social-rotator.html`
- **Width:** 400, **Height:** 120
- **Custom CSS:** None required (widget handles all styling)
- **Shutdown when not visible:** Optional — widget reconnects when source becomes active
- **Audio:** None (widget is silent)

---

## 10. Extensibility Plan

This Phase 1 architecture is intentionally future-proof:

- **New widget:** Add `public/widgets/<widget-name>.html`, add config section to `config.json`, add dashboard UI section. Backend needs zero changes.
- **New platform API (Twitch chat, YouTube subs):** Add connector file in `src/backend/connectors/`. Register in `server.js`. Push data via same WebSocket hub.
- **Database migration:** Replace `config-manager.js` file I/O with SQLite/PostgreSQL. All other code unchanged.
- **Hosted platform:** Move backend to cloud. Widgets point to cloud URL. Dashboard becomes a SaaS web app.

---

## 11. Success Criteria

- [ ] Dashboard loads at `localhost:3000/dashboard`
- [ ] Streamer can add/remove platforms and set rotation speed
- [ ] Widget displays in Lumia/OBS and rotates through platforms
- [ ] Live preview in dashboard updates instantly when config changes
- [ ] Theme presets (Sleek, Bold, Retro, Animated) all render correctly
- [ ] Widget reconnects automatically if backend restarts
- [ ] No external API keys needed for this phase (backend is config-only)

---

## 12. Out of Scope (Future Phases)

- External platform API integration (Twitch EventSub, YouTube Data API, etc.)
- Additional widgets (goal bar, chat hype, supporters ticker, etc.)
- Authentication / multi-user support
- Cloud hosting / SaaS platform
- Analytics / metrics

---

## 13. Open Questions / Assumptions

1. **Assumption:** Lumia Stream browser source supports modern CSS (animations, custom properties, WebSocket). If not, we may need fallbacks.
2. **Assumption:** Streamer's laptop can run Node.js + Lumia + OBS simultaneously without performance issues.
3. **Decision:** Built-in platform icons will use inline SVGs or Font Awesome CDN for Phase 1. Custom platforms require an icon URL.
4. **Decision:** `.env` file stays empty for Phase 1 since no external APIs are called. It exists as structural scaffolding for Phase 2.
