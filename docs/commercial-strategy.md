# ManageFC — Commercial Strategy & Recommendations

## 1. Vision

Evolve ManageFC from a single-team tracker into a multi-tenant SaaS platform for managing football clubs, teams, and age groups. One platform serving multiple clubs, each with multiple teams across age groups from Under 7 to Under 18.

---

## 2. Multi-Tenant Data Model

### Proposed Hierarchy
```
Club
  └── Team (e.g. "SUMAS Boys")
        └── Age Group (U7 – U18)
              ├── Players
              └── Matches
                    ├── Events (goals, assists)
                    └── Poll (parent voting)
```

### Key Entities
| Entity | Fields |
|---|---|
| Club | name, adminPin, subscription tier, created date |
| Team | clubId, name, ageGroup, coachPin |
| Player | teamId, name, jerseyNumber |
| Match | teamId, opponent, date, venue, homeAway, competition, status, events, poll, report |

---

## 3. Recommended Architecture — Option 2 (Next.js + Supabase)

### Why This Stack
- **Next.js** on Vercel — server-side rendering, fast initial load, PWA support, edge CDN
- **Supabase** — Postgres database (relational, suits clubs/teams/players naturally), built-in Auth, Realtime WebSockets for live match updates, Row Level Security for multi-tenant data isolation
- **Stripe** — subscription billing per club
- **Resend** — transactional email (auth, notifications)
- **Sentry** — error monitoring

### System Architecture
```
┌──────────────────────────────────────────────────────┐
│   Next.js PWA  (Vercel — global edge CDN)            │
└──────────────────┬───────────────────────────────────┘
                   │ REST + WebSockets
         ┌─────────┴──────────┐
         ▼                    ▼
    Supabase               Supabase
    (Postgres DB +         Realtime
     Auth + RLS)           (live scores)
         │
         ▼
      Stripe
  (per-club billing)
```

### Why Supabase Over Firebase
| Factor | Firebase | Supabase |
|---|---|---|
| Data model | Document (NoSQL) | Relational (Postgres) |
| Multi-tenancy | Manual scoping | Native Row Level Security |
| Cost at scale | Expensive (per read) | Cheaper (row-based) |
| Vendor lock-in | High | Low (open source, self-hostable) |
| Real-time | Yes | Yes |
| Auth | Yes | Yes |

---

## 4. Operating Costs

### Stage 1 — MVP (0–50 clubs)

| Service | Plan | Monthly Cost |
|---|---|---|
| Supabase | Pro (1 project) | ~£20 |
| Vercel | Pro | ~£16 |
| Domain | Amortised | ~£1 |
| Email (Resend) | Free tier | £0 |
| Stripe | 1.4% + 20p per transaction | % of revenue |
| **Total fixed** | | **~£37/month** |

Breaks even at approximately **4 paying clubs** at £10/club/month.

### Stage 2 — Growth (50–200 clubs)

| Service | Plan | Monthly Cost |
|---|---|---|
| Supabase | Pro + compute add-ons | ~£47 |
| Vercel | Pro | ~£16 |
| Email (Resend) | Starter | ~£16 |
| Monitoring (Sentry) | Team | ~£20 |
| **Total fixed** | | **~£100/month** |

### Stage 3 — Scale (200–500 clubs)

| Service | Plan | Monthly Cost |
|---|---|---|
| Supabase | Pro + larger compute | ~£118 |
| Vercel | Pro team | ~£39 |
| Email (Resend) | Business | ~£39 |
| Monitoring (Sentry) | Business | ~£63 |
| Backups / Storage | Supabase add-on | ~£20 |
| **Total fixed** | | **~£280/month** |

### Revenue vs Cost Summary

| Stage | Clubs | Fixed Infra | Revenue (£10/club) | Estimated Profit |
|---|---|---|---|---|
| MVP | 0–50 | ~£37/mo | £0–£500 | Breaks even at ~4 clubs |
| Growth | 50–200 | ~£100/mo | £500–£2,000 | ~£1,349/mo at 150 clubs |
| Scale | 200–500 | ~£280/mo | £2,000–£5,000 | ~£3,584/mo at 400 clubs |

*Stripe fees (~1.4% + 20p per transaction) deducted from revenue figures above.*

### Supabase Capacity vs Projected Usage

| Resource | Pro Limit | Estimated at 200 clubs |
|---|---|---|
| Database size | 8GB | ~500MB |
| Bandwidth | 100GB | ~15GB |
| Monthly Active Users | 100,000 | ~18,000 |
| Concurrent Realtime connections | 500 | ~300 peak |

---

## 5. Development Strategy — Using Claude AI

### Why Claude Instead of Hiring a Developer

| Factor | Hiring Developer | Using Claude |
|---|---|---|
| Development cost | £15,000–£30,000 | ~£100–£200/month (subscription) |
| Availability | Business hours | 24/7 |
| Timeline | 3–6 months | 3–6 months (at your pace) |
| Domain knowledge | Leaves with the developer | Stays in your codebase and docs |
| Iteration speed | Slow (back and forth) | Immediate |

The £30,000 developer budget becomes runway and marketing budget instead.

### What Claude Can Handle

| Task | Capability |
|---|---|
| Architecture and database design | Excellent |
| Writing all application code | Excellent |
| Supabase schema and RLS policies | Excellent |
| Next.js frontend components | Excellent |
| Stripe integration | Excellent |
| PWA / offline support | Excellent |
| Writing tests | Good |
| Debugging errors | Good |
| GDPR compliance implementation | Good |
| DevOps / deployment configuration | Good |

### What You Need to Provide

| Input | Why |
|---|---|
| Product decisions | Features, pricing, UX flow |
| Device testing | Claude cannot click buttons or use a browser |
| Account setup | Supabase, Vercel, Stripe — you create the accounts |
| Reviewing and approving changes | You are the product owner |
| Real-world feedback | From coaches and parents using it |

### Working Effectively With Claude

- Keep a `CLAUDE.md` file in the project with key decisions, conventions, and context
- Commit to git frequently — work is never lost between sessions
- Paste error messages back when things go wrong — Claude can diagnose them
- Build one phase at a time — don't try to build everything at once

---

## 6. Build Phases

### Phase 1 — Foundation (2–3 weeks)
- Supabase schema: clubs, teams, age groups, players, matches
- Auth: email login, role system (club admin, coach, parent)
- Basic club and team management UI

### Phase 2 — Core Features (3–4 weeks)
- Port existing match tracking to multi-team
- Live score, goals, events per team
- Parent view per team

### Phase 3 — Commercial Layer (2–3 weeks)
- Stripe subscriptions per club
- Free tier (1 team), paid tier (unlimited teams)
- Club admin dashboard

### Phase 4 — Polish (ongoing)
- Leaderboards, reports, analytics
- Mobile PWA optimisation
- GDPR controls and children's data compliance

---

## 7. Monetisation

### Suggested Pricing Tiers

| Tier | Price | Includes |
|---|---|---|
| Free | £0/month | 1 team, basic match tracking |
| Club | £10/month | Up to 5 teams, all features |
| Academy | £25/month | Unlimited teams, analytics, priority support |

### Revenue Potential

| Clubs | Average Revenue/Club | Monthly Revenue | Annual Revenue |
|---|---|---|---|
| 50 | £10 | £500 | £6,000 |
| 200 | £10 | £2,000 | £24,000 |
| 500 | £15 | £7,500 | £90,000 |

---

## 8. Compliance Considerations

| Area | Requirement |
|---|---|
| GDPR | Required — EU/UK data protection law applies |
| Children's data | Extra protections required for under-18 player data |
| Privacy policy | Must be in place before launch |
| Terms & conditions | Must be in place before launch |
| Data deletion | Users must be able to request deletion of their data |
| Parental consent | Required for storing data about players under 13 |

Estimated legal setup cost: £1,500–£3,000 (one-off).

---

## 9. One-Off Setup Costs

| Item | Estimated Cost |
|---|---|
| Claude AI subscription (ongoing) | ~£100–£200/month |
| Domain name | ~£10–£15/year |
| Legal (T&Cs, privacy policy, GDPR) | £1,500–£3,000 |
| Design / branding | £0 if DIY, £2,000–£5,000 if outsourced |
| Apple Developer account (if going native) | £99/year |
| Google Play account (if going native) | £20 one-off |

---

## 10. Recommended Next Steps

1. Validate the product with 2–3 local clubs before building the commercial version
2. Start Phase 1 with Claude — Supabase schema and auth
3. Register a domain and create accounts on Supabase, Vercel, and Stripe
4. Set up legal documentation in parallel with development
5. Launch free tier first to build user base, introduce paid tiers once validated

---

*Document prepared: March 2026*
*Based on ManageFC codebase at SW v48*
