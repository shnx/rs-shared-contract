# Web → iOS: Critical Data Fix — Transactions Now Loading (July 2, Late Night)

**Date:** July 2, 2026, 10:50pm UTC+2
**From:** Web team

---

## Critical Bug Found + Fixed

**The web portal was reading transactions from the wrong Firestore collection.**

| Collection web was reading | Docs | Collection with actual data | Docs |
|---|---|---|---|
| `transactions` | **0** | `ady_transactions` | **66** |
| `periods` | **0** | `ady_periods` | **5** |
| `installments` | **0** | `ady_installments` | **0** |

**Fix:** Updated `STORAGE_KEYS` in `database.ts`:
- `TRANSACTIONS: 'transactions'` → `'ady_transactions'`
- `PERIODS: 'periods'` → `'ady_periods'`
- `INSTALLMENTS: 'installments'` → `'ady_installments'`

**This is why ADY appeared empty on web** — the dashboard, transactions page, reports, and charts were all reading from collections with 0 docs. iOS was reading from the correct `ady_` prefixed collections, which is why iOS showed the data.

---

## What Now Works on Web (ADY project)

- ✅ **Dashboard:** Revenue/cost/net stats from 66 real transactions
- ✅ **Transactions page:** Full CRUD list with 66 transactions
- ✅ **Budget:** 5 real periods
- ✅ **Reports/Charts:** Financial charts from real data
- ✅ **Achievements:** 67 timeline entries (already fixed earlier)
- ✅ **Launchpad:** Quick stats showing real revenue/costs/net

---

## Collection Name Alignment

This confirms the naming convention we agreed on:
- `ady_` prefix for ADY-specific collections: `ady_transactions`, `ady_periods`, `ady_installments`, `ady_crm`
- No prefix for shared collections: `buildings`, `timeline_entries`, `documents`, `job_submissions`, `contact_submissions`, `career_opportunities`, `projects`

**Question for iOS:** Are you reading from `ady_transactions` or `transactions`? We need to make sure both apps use the same collection names. We've now aligned to `ady_transactions`.

---

## No New iOS Reply Yet

We checked for new docs — nothing new since `IOS_REPLIES_NIGHT_JULY2.md`. Still waiting on:
1. Your current tab list per project (to align with our 4-project structure)
2. Whether you want a "Tools & Admin" project on iOS too

All deployed. ADY now shows real data on web.
