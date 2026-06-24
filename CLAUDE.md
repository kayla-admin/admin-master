# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**Admin Master** is a single-page personal admin dashboard for Kayla Morkel at Bob Group. It is a single HTML file (`dashboard.html`) deployed via GitHub Pages with no build step, no dependencies, and no package manager. All logic, styles, and markup are self-contained in that one file.

## Deployment

- Hosted on **GitHub Pages** (no build step — just push to `main`)
- To deploy changes: commit and push to `main`; GitHub Pages serves the updated file automatically
- Live URL is served directly from the repo root
- Git identity for this repo: `Kayla Morkel <kaylaannebambi@gmail.com>` (configured locally — not global)

## Architecture

The entire app lives in `dashboard.html`. Its structure:

1. **CSS** — inline `<style>` block using CSS custom properties (`--bg`, `--surface`, `--people`, `--expenses`, etc.) for theming. Dark-mode only.
2. **Firebase** — loaded via CDN (v9 compat SDK). Three services used:
   - `firebase-auth-compat` — Google OAuth, restricted to `kayla@bob.co.za` via `hd` domain hint and an email check on `onAuthStateChanged`
   - `firebase-firestore-compat` — single Firestore document (`dashboard/state`) holds all app state
3. **State** — `memState` is the in-memory mirror of the Firestore document. All writes go through `saveFS()` which calls `docRef.set(fs)` (full document overwrite). Firestore's `onSnapshot` keeps `memState` live.
4. **Rendering** — no framework; all UI is built by string-concatenating HTML in `renderAll()` → `renderSummary()`, `renderWorklog()`, `renderTasks()`, `renderGrid()`, `renderTravelClaims()`. Every save triggers a full re-render with scroll/expand state manually preserved.
5. **Categories** — the `categories` array (line ~521) defines all recurring task sections (People, Expenses, Leadership, Health & Safety, Facilities) with their tasks, subtasks, and reset frequencies (`daily` / `weekly` / `monthly` / `quarterly` / `annual` / `manual`).

## Key data shapes (inside `memState` / Firestore `dashboard/state`)

```
tasks         — { [key]: { period, done, created, completedTs } }  — recurring task state
onboardings   — [{ id, name, location, startDate, steps[], archived, created }]
offboardings  — [{ id, name, location, lastDay, steps[], archived, created }]
projects      — [{ id, name, steps[], archived, created, completedTs }]  — ad-hoc tasks
worklog       — [{ id, name, category, created, completed }]
travelLog     — [{ id, date, from, to, notes }]  — legacy flat list
travelBatches — [{ id, trips[], submitted, submittedTs, label, created }]  — current model
```

## Firebase config

The Firebase project config (API key, project ID, etc.) is hardcoded in the HTML at line ~313. This is intentional for a GitHub Pages app. The `ALLOWED_EMAIL` constant (`kayla@bob.co.za`) is enforced on `onAuthStateChanged` — any other Google account is immediately signed out. This has been verified: accounts outside `bob.co.za` cannot authenticate.

Firestore security rules enforce access server-side as a second layer — even if the client JS were modified, Firestore rejects any request whose auth token isn't `kayla@bob.co.za`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null
        && request.auth.token.email == 'kayla@bob.co.za';
    }
  }
}
```

The `/{document=**}` wildcard covers all current and future collections.

## Onboarding / offboarding steps

`ONBOARDING_STEPS` and `OFFBOARDING_STEPS` arrays (lines ~341–375) are Kayla's actual process. Steps with `menlyn: true` or `bryanston: true` are filtered by `buildSteps()` based on the employee's office location. Steps with `locationNote: true` have the location appended to the step text at creation time.

## Travel claims

Travel uses a batch system: trips accumulate in an "active batch" (unsubmitted). Submitting marks the batch done and starts a new one. The Deel submission deadlines are the 15th (soft) and 19th (hard) of each month — the UI shows alerts based on `day` from `new Date()`.

## Making changes

- Edit `dashboard.html` directly — there is no compilation or transpilation
- CSS variables are at the top of the `<style>` block; add new colour tokens there if needed
- To add a recurring task: add an entry to the relevant `categories[].tasks` array
- To add an onboarding/offboarding step: edit `ONBOARDING_STEPS` or `OFFBOARDING_STEPS` (existing instances in Firestore are not affected — only new ones pick up the new template)
- `uid()` generates short random IDs for new records
