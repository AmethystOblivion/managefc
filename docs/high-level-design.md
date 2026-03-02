# High-Level Design Document — ManageFC

## 1. Purpose

ManageFC enables a football coach to record live match data (score, goals, assists, MOTM) during games, while parents follow along in real time and participate in player-of-the-match voting.

---

## 2. User Roles

| Role | Access | Authentication |
|---|---|---|
| Coach | Full read/write; all tabs; admin menu | PIN: `SUMAS11` |
| Parent | Read-only; matches + leaderboard + voting | No PIN required |

---

## 3. Application Tabs

| Tab | Coach | Parent | Key Content |
|---|---|---|---|
| **Players** | Add/remove/rename players | View roster | Jersey tiles |
| **Matches** | Full match management | Live score, report, vote | Score banner, events, controls |
| **Leaderboard** | View stats | View stats | Goals / Assists / C-POTM / P-POTM |
| **Votes** | Full vote management | Hidden | Voter breakdown per match |

---

## 4. Match Lifecycle

```
pending  ──► live ──► ht ──► live ──► ended
                  [HT]      [2nd]     [FT]
                                        │
                                  poll auto-opens
```

| State | Timer | Coach Controls | Parent Sees |
|---|---|---|---|
| `pending` | Off | Kick-off | Pending |
| `live` (1st) | Running | HT | Live + timer |
| `ht` | Paused | 2nd Half | Half Time |
| `live` (2nd) | Running | FT | Live + 2nd Half badge |
| `ended` | Stopped | — | Full Time, C-POTM, Poll |

---

## 5. Goal Recording

```
Coach selects player
    │
    ├── Optional: select assist provider
    ├── Optional: override goal time (live timer shown, editable)
    └── Optional: mark as Penalty (disables assist dropdown)
            │
            ▼
  Event stored: { player, type, assistBy, time, penalty }
  Scorer lines appear under SUMAS label on banner
```

---

## 6. Leaderboard Computation

All stats are derived **live from match data** — not from player counters.

| Category | Source |
|---|---|
| Goals | `match.events` where `type === 'goal'` |
| Assists | `event.assistBy` on goals + standalone assist events |
| C-POTM | `match.motm` field (coach selection) |
| P-POTM | Wins from **closed** polls only (by highest vote count) |

---

## 7. Parent Voting (P-POTM Poll)

```
Match ends ──► Poll auto-opens
                    │
              Parent selects player
              Parent enters name
              Vote recorded with unique voter ID
                    │
              Coach closes poll ──► Winner locked in
                                  ──► Counts toward P-POTM leaderboard
```

- Parents can change or remove their vote while poll is open
- Coaches can remove any individual vote
- Tied winners both receive a win credit on the leaderboard
- Only **closed** polls count toward leaderboard standings

---

## 8. Match Info

Each match stores: `date` (ISO, calendar picker), `homeAway`, `venue`, `competition`.

- **Home** → dropdown: Charvil / Cantley / Enter venue
- **Away** → free text
- Displayed as icon row on match banner for all users
- History tabs sorted chronologically by date; most recent match auto-selected as LATEST

---

## 9. Key Design Decisions

| Decision | Rationale |
|---|---|
| Single HTML file | Zero build tooling; deployable anywhere; easy to fork |
| Firestore as single source of truth | Real-time sync across all devices with no backend code |
| Stats derived from events, not counters | Prevents drift between raw data and displayed totals |
| PIN in client-side JS | Acceptable for a private club app; no sensitive PII stored |
| Versioned service worker cache | Guarantees stale cache is busted on every deploy |
| `latestIdx` by date, not array order | Matches added out of order still surface correctly |

---

## 10. Major Functions Reference

| Function | Purpose |
|---|---|
| `getLatestMatchIdx()` | Finds match index with most recent date |
| `computeStatsFromMatches()` | Derives all leaderboard stats from match events |
| `setMatchStatus(idx, status)` | Transitions match state machine; triggers timer and poll |
| `tickMatchTimer()` | Updates live timer display every second |
| `addMatchEvent(idx, type)` | Records goal or assist with time, assist, penalty |
| `removeMatchEvent(idx, eventIdx)` | Removes event and reverses player stat credit |
| `isEditable()` | Single gate for all write operations (coach or override) |
| `renderMatches()` | Full match section render (tabs + banner + forms) |
| `computeStatsFromMatches()` | Aggregates goals, assists, C-POTM, P-POTM from raw data |
| `confirmVote()` | Records parent poll vote with deduplication |
| `togglePollClosed(idx)` | Coach opens/closes parent voting window |
| `saveMatches()` | Persists entire matches array to Firestore |
| `esc(str)` | HTML-escapes all user-supplied strings (XSS prevention) |
