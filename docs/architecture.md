# Architecture Document — ManageFC

## 1. Overview

ManageFC is a single-page, progressive web application (PWA) for managing youth football match statistics. It is designed to be used by two distinct roles — **coaches** and **parents** — with real-time shared state backed by Google Firebase Firestore. There is no server-side code; all logic runs in the browser.

---

## 2. Technology Stack

| Layer | Technology | Version / Detail |
|---|---|---|
| UI | Vanilla HTML/CSS/JavaScript (ES6+) | No framework |
| Database | Google Firebase Firestore | Compat SDK v10.12.0 |
| Offline / PWA | Service Worker + Web App Manifest | Cache-first offline fallback |
| Hosting | GitHub Pages | `main` branch root |
| External link | GotSport league table | Outbound link only, no API |

---

## 3. System Architecture

```
┌─────────────────────────────────────────┐
│              Browser (Client)           │
│                                         │
│  ┌──────────────┐   ┌────────────────┐  │
│  │  index.html  │   │    sw.js       │  │
│  │  (all logic) │   │ (cache/offline)│  │
│  └──────┬───────┘   └────────────────┘  │
│         │ onSnapshot / set()            │
│         ▼                               │
│  ┌──────────────────────────────────┐   │
│  │     Firebase Firestore           │   │
│  │   collection: data               │   │
│  │   ├── doc: players               │   │
│  │   └── doc: matches               │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**All application state lives in Firestore.** The browser holds in-memory copies (`players[]`, `matches[]`) that are kept in sync via real-time `onSnapshot` listeners. Every mutation is immediately written back to Firestore with `set()`.

---

## 4. Data Storage

### Firebase Firestore

| Collection | Document | Field | Format |
|---|---|---|---|
| `data` | `players` | `list` | `JSON.stringify(Player[])` |
| `data` | `matches` | `list` | `JSON.stringify(Match[])` |

Both documents store their entire payload as a single JSON string. This is a **document-per-entity-type** pattern — there are no sub-collections.

### Local Storage

| Key | Purpose |
|---|---|
| `sumasVoterId` | Unique voter identifier (persists across sessions) |

### Session Storage

| Key | Purpose |
|---|---|
| `sumasRole` | `'coach'` or `'parent'` for current session |
| `sumasUnlocked` | `'true'` if coach PIN was entered |

---

## 5. Data Model

### Player
```
{ name, goals, assists, motm }
```

### Match
```
{
  opponent, date, homeAway, venue, competition,
  scoreSumas, scoreOpp, status,
  elapsedMs, liveStart, htReached,
  events[], motm,
  report, reportPublished,
  poll: { opened, closed, votes{}, voters[], voterNames[], voterChoices{} }
}
```

### Event (inside `match.events[]`)
```
// Goal
{ player, type:'goal', count, assistBy, time, penalty }

// Standalone assist (legacy)
{ player, type:'assist', count }
```

---

## 6. Authentication & Authorisation

```
Page Load
    │
    ▼
Role Selection Overlay
    ├── Parent ──────────────────────────► Read-only access
    └── Coach ──► PIN modal (SUMAS11) ──► Full edit access
                      │
                      └── Override modal (WT11) ──► 5-min timed edit access
```

- **No server-side auth** — roles are stored in `sessionStorage` only
- **Override code** (`WT11`) allows a temporary (5 min) coach-level session without the PIN, intended for pitch-side emergencies
- `isEditable()` is the single gate function checked before all write operations

---

## 7. PWA & Offline

**Service Worker strategy: Network-first, cache fallback**

1. On fetch: attempt network
2. On success: clone response into cache, return live response
3. On failure: return cached version

Assets cached: `index.html`, `manifest.json`, root (`./`)

Firebase calls are **not** intercepted by the SW — if offline, Firestore reads/writes silently fail. The app degrades to a read-only view of whatever was last loaded into memory.

Cache is versioned (`sumas-stats-vN`). Every deployment increments N, which causes the SW activate handler to purge all older cache versions.

---

## 8. Real-time Sync

```
Coach action (e.g. add goal)
        │
        ▼
  In-memory update (matches[])
        │
        ▼
  saveMatches() → Firestore set()
        │
        ▼
  onSnapshot fires on ALL connected clients
        │
        ▼
  renderMatches() on every device simultaneously
```

All users (coaches and parents on any device) see updates within ~1 second with no polling.
