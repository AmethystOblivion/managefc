# CLAUDE.md — ManageFC Project Context

This file gives Claude the context needed to work effectively on this project across sessions.
Read this at the start of every session before making any changes.

---

## Project Overview

ManageFC is a football statistics PWA, forked from a single-team tracker (SUMAS FC) and being evolved into a commercial multi-tenant platform for managing multiple clubs, teams, and age groups.

**Current state:** Single-team PWA, fully functional, deployed on GitHub Pages.
**Target state:** Multi-tenant SaaS supporting multiple clubs, teams (U7–U18), coaches, and parents.

---

## Repository

- **Local:** `/Users/anilrao/managefc/`
- **GitHub:** `https://github.com/AmethystOblivion/managefc`
- **Live URL:** `https://amethystoblivion.github.io/managefc/`
- **Branch:** `main`

---

## Current Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Single HTML file (`index.html`) — vanilla JS, no framework |
| Database | Google Firebase Firestore (compat SDK v10.12.0) |
| Auth | Client-side PIN (sessionStorage) |
| Offline | Service worker (`sw.js`) — network-first, cache fallback |
| Hosting | GitHub Pages |

---

## Planned Commercial Stack (Next Phase)

| Layer | Technology |
|---|---|
| Frontend | Next.js (React) PWA on Vercel |
| Database | Supabase (Postgres + Row Level Security) |
| Realtime | Supabase Realtime (WebSockets) |
| Auth | Supabase Auth (email/password, Google) |
| Billing | Stripe (per-club subscriptions) |
| Email | Resend |
| Monitoring | Sentry |
| Hosting | Vercel |

---

## Current Data Model

### Player
```javascript
{ name, goals, assists, motm }
```

### Match
```javascript
{
  opponent, date, homeAway, venue, competition,
  scoreSumas, scoreOpp, status,           // status: pending | live | ht | ended
  elapsedMs, liveStart, htReached,        // timer fields
  events[],                               // goal/assist events
  motm,                                   // coach POTM selection
  report, reportPublished,                // match report
  poll: { opened, closed, votes{}, voters[], voterNames[], voterChoices{} }
}
```

### Event
```javascript
// Goal
{ player, type:'goal', count:1, assistBy, time, penalty }
// Assist (legacy)
{ player, type:'assist', count }
```

---

## Planned Multi-Tenant Data Model (Supabase)

```
clubs          — id, name, admin_pin, subscription_tier, created_at
teams          — id, club_id, name, age_group (U7–U18), coach_pin
players        — id, team_id, name, jersey_number
matches        — id, team_id, opponent, date, home_away, venue, competition, status, ...
match_events   — id, match_id, player_id, type, assist_by, time, penalty
polls          — id, match_id, opened, closed
poll_votes     — id, poll_id, voter_id, player_id, voter_name
```

---

## User Roles

### Current
| Role | Auth | Access |
|---|---|---|
| Coach | PIN: stored in sessionStorage | Full edit |
| Parent | No PIN | Read-only + voting |

### Planned Commercial
| Role | Auth | Access |
|---|---|---|
| Club Admin | Email/password (Supabase Auth) | Manage all teams in club |
| Coach | Email/password | Full edit for assigned team |
| Parent | Email/password or magic link | Read-only + voting for assigned team |

---

## Key Functions (Current Codebase)

| Function | Purpose |
|---|---|
| `getLatestMatchIdx()` | Finds match with most recent date |
| `computeStatsFromMatches()` | Derives leaderboard stats from match events |
| `setMatchStatus(idx, status)` | Match state machine — pending→live→ht→ended |
| `tickMatchTimer()` | Updates live match timer every second |
| `addMatchEvent(idx, type)` | Records goal/assist with time, assist, penalty |
| `isEditable()` | Gate for all write operations |
| `saveMatches()` | Persists matches array to Firestore |
| `esc(str)` | HTML-escapes all user strings (XSS prevention) |

---

## Service Worker

- Cache name versioned as `sumas-stats-vN`
- **Must increment N on every deployment** — this busts the cache for all users
- Current version: v48
- File: `sw.js`

---

## Coding Conventions

- Always use `esc()` on any user-supplied string rendered into HTML (XSS prevention)
- All stats must be derived from `match.events` — never rely on `player.goals` / `player.assists` counters alone (they can drift)
- Bump SW cache version on every commit that changes `index.html`
- Commit and push after every feature — never leave work uncommitted
- `latestIdx` is determined by most recent `match.date` (ISO format), not array position

---

## Home Venues

Preset home venue options:
- Charvil
- Cantley
- (custom free text)

---

## Jersey Numbers

```javascript
const jerseyNumbers = {
    'sammy': 8, 'adam': 1, 'jayden': 26, 'kian': 10,
    'isaac': 5, 'louie': 18, 'rylee': 16, 'will': 14,
    'euan': 9, 'james': 17, 'jude': 12, 'luke': 18,
    'noah': 4, 'oscar': 6, 'wilbur': 2, 'mali': 3
};
```

---

## Build Phases (Commercial Roadmap)

| Phase | Scope | Status |
|---|---|---|
| Phase 1 | Supabase schema, Auth, club/team management UI | Not started |
| Phase 2 | Port match tracking to multi-team | Not started |
| Phase 3 | Stripe billing, free/paid tiers, club admin dashboard | Not started |
| Phase 4 | Leaderboards, reports, analytics, GDPR controls | Not started |

---

## Commercial Strategy

- **Target market:** Youth football clubs in the UK
- **Pricing model:** Freemium — 1 team free, paid tiers for more teams
- **Suggested tiers:**
  - Free: 1 team, basic match tracking
  - Club (£10/month): up to 5 teams, all features
  - Academy (£25/month): unlimited teams, analytics, priority support
- Full strategy doc: `docs/commercial-strategy.md` (local only, not in git)

---

## Docs (Local Only — Not in Git)

All files in `docs/` are excluded from git via `.gitignore`.

| File | Contents |
|---|---|
| `docs/architecture.md` | Full technical architecture of current app |
| `docs/high-level-design.md` | Feature design and system behaviour |
| `docs/commercial-strategy.md` | Multi-tenant vision, costs, build phases, monetisation |

---

## Important Reminders for Claude

- Docs folder is gitignored — never try to commit anything under `docs/`
- Do not expose PIN codes or Firebase config in any committed file
- Always read a file before editing it
- Always bump `sw.js` cache version after changing `index.html`
- Always push after committing — `git push origin main`
- The project is a PWA — test offline behaviour when making SW changes
- GDPR compliance is required — players are minors (under 18)
