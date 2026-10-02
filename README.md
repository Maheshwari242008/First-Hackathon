# Privacy-First Disease Outbreak Early-Warning System

> Lab case counts in, explainable neighbourhood risk out. No patient data, ever.

**Status:** Prototype risk indicator. Not a validated medical prediction and not a diagnosis.

---

## Overview

Outbreaks of diseases such as dengue, malaria, chikungunya, typhoid, cholera and diarrhoea usually become visible to the public only after they have already grown. Laboratories see the early signal, but that data rarely reaches residents, and sharing it raises serious privacy concerns.

This project closes that gap. Laboratories submit **aggregate case counts by area, never patient details**. The backend aggregates them into roughly 1 km grid cells and suppresses any cell with fewer than 3 cases so nobody can be re-identified. A transparent scoring engine rates each cell **LOW, MODERATE or HIGH**. Residents get a plain-language alert with prevention steps; health authorities get a hotspot dashboard.

**Core idea:** a privacy gate (small-number suppression) combined with an explainable risk score that shows exactly why an area is flagged.

## Features

- Lab case entry (disease, date, area, count only)
- Privacy gate: cells with fewer than 3 recent cases are hidden from public views
- Explainable risk score (volume, growth, neighbour spillover)
- Risk map with coloured cells and disease filters
- Resident home: current area risk, alert detail, prevention tips
- Push notifications plus an in-app alert feed
- Authority dashboard: cases by area, 14-day trend, alert list, replay slider
- Role-based access: citizen, lab, authority

## Tech stack

| Area | Choice |
|---|---|
| Mobile app | React Native, Expo, TypeScript |
| Navigation | React Navigation |
| Data fetching | TanStack Query |
| Backend / database | Supabase (PostgreSQL, PostGIS, Edge Functions, Realtime) |
| Authentication | Supabase Auth |
| Maps | Mapbox or Google Maps |
| Notifications | Expo Notifications |
| Version control | Git and GitHub |

## How it works

```
Lab -> submit-report (Edge Function) -> lab_reports (private)
    -> aggregation -> area_disease_stats
    -> compute_risk -> risk_results
    -> alerts -> push notifications
    -> Citizen app (aggregates only)
```

1. **Ingest:** only whitelisted fields are accepted; anything else is discarded.
2. **Aggregate:** counts are grouped by area and day.
3. **Privacy gate:** areas with fewer than 3 recent cases return `NONE`.
4. **Score:** the risk engine computes a score for each area and disease.
5. **Communicate:** map, alerts and notifications with prevention advice.

### Risk scoring

For each (area, disease) pair, the engine compares two consecutive 7-day windows.

```
recent   = cases in last 7 days
prior    = cases in the 7 days before that
growth   = (recent - prior) / max(prior, 1)
neighbor = recent cases in the 8 adjacent cells

if recent < 3: level = NONE

score = 0.5 * min(recent / 15, 1)
      + 0.3 * min(max(growth, 0) / 2, 1)
      + 0.2 * min(neighbor / 20, 1)
score *= severity[disease]
```

| Level | Score |
|---|---|
| NONE | Suppressed (fewer than 3 recent cases) |
| LOW | below 0.25 |
| MODERATE | 0.25 to below 0.55 |
| HIGH | 0.55 or above |

Thresholds and severity weights are unvalidated and for demonstration only.

## Privacy by design

- No patient fields exist in the schema (no name, age, address, phone or exact coordinates).
- `lab_reports` is private. Citizens can only read aggregated views and RPCs.
- Row Level Security is enabled on every table.
- Server-side field whitelisting on every submission.
- Alerts are per grid cell, never a radius around a case.
- A 1 km grid with a minimum of 3 cases reduces, but does not eliminate, re-identification risk.

## Database overview

| Table | Purpose |
|---|---|
| `profiles` | User role, home area, notification settings |
| `diseases` | Names, severity, symptoms, precautions |
| `areas` | Grid cells with location |
| `lab_reports` | Private lab submissions (aggregate counts) |
| `area_disease_stats` | Daily aggregated counts |
| `risk_results` | Latest level, score and explanation per area and disease |
| `alerts` | Generated alerts shown to residents |
| `push_log` | Record of notifications sent |

## Project structure

```
/app                  React Native (Expo) mobile app
/supabase
  /migrations         Database schema and RLS policies
  /functions          Edge Functions (submit-report, send-alerts)
  seed.sql            Synthetic seed data
/docs                 Contract, diagrams, notes
README.md
```

## Getting started

### Prerequisites

- Node.js 18 or later
- Expo CLI
- Supabase CLI
- A Supabase project

### Setup

```bash
git clone <repo-url>
cd <repo-name>

# Mobile app
cd app
npm install
cp .env.example .env
npx expo start
```

Add these to `app/.env`:

```
EXPO_PUBLIC_SUPABASE_URL=your-project-url
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

### Backend

```bash
supabase login
supabase link --project-ref <project-ref>
supabase db push
supabase functions deploy submit-report
supabase functions deploy send-alerts
```

Then load the synthetic data from `supabase/seed.sql`.

> Never commit the Supabase **service role key**. It belongs only in Edge Function secrets. The app uses the anon key.

## Demo flow

1. Show the map: mostly green, one dengue cluster at MODERATE.
2. A lab enters 4 new dengue cases in a nearby cell.
3. The privacy gate and aggregation run; the cell turns HIGH.
4. The "why this alert" panel updates.
5. The resident view shows a notification with prevention tips.
6. Show neighbour spillover and the 14-day trend.

## Testing checklist

- [ ] A citizen cannot read `lab_reports`
- [ ] A lab cannot read another lab's rows
- [ ] A submission with an extra name field is stripped or rejected
- [ ] A cell with 2 recent cases returns `NONE`
- [ ] A cell with 3 recent cases is scored normally
- [ ] `prior = 0` causes no divide-by-zero
- [ ] Worked example (11 recent, 5 prior, 8 neighbour cases, dengue) scores about 0.63, HIGH
- [ ] An alert is created once per level change, not on every recompute

## Limitations

- Seed data is synthetic.
- Not a validated medical prediction; never use it for diagnosis.
- False alarms cause anxiety and missed alarms cause harm; a real deployment needs health-authority oversight and epidemiological validation of thresholds.

## Future scope

Real lab integrations, SMS alerts, anomaly detection (EWMA, CUSUM), spatial clustering, forecasting, differential-privacy noise, symptom self-reporting, multilingual alerts, and weather or wastewater signals.

## Team

| Name | Role |
|---|---|
| [Your name] | Backend |
| Dipika | |
| Shirisha | |
| Shravanti | |
| Maheshwari | |

## Contributing

1. Never push directly to `main`.
2. Create a branch per task, open a pull request, and get one review.
3. Post errors in `#sos` and attend the daily standup in `#standup`.
4. Never add patient-level data to the database, code or screenshots.

---

*Last updated: 2 October 2026*