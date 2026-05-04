# Cycle Tracker — Project Context

## What it is
A period/menstrual cycle and ovulation tracker built as a **single-file HTML web app** (`index.html`). No framework, no build step, no backend. All logic is vanilla JS/CSS embedded in the file.

**Live URL:** https://gamachem-arch.github.io/cycle-tracker/
**Repo:** https://github.com/gamachem-arch/cycle-tracker
**Local path:** `/Users/michel/Claude/Period Tracker/`

---

## Tech decisions

| Decision | Choice | Reason |
|---|---|---|
| Stack | Single HTML file | Zero deployment friction, no dependencies |
| Hosting | GitHub Pages | Free, auto-deploys on push to `main` |
| Storage | `localStorage` (key: `ct_v1`) | No backend needed, all data stays on device |
| Auth | SSH key (`~/.ssh/id_ed25519`) | Set up during this session for git push |
| Dev server | `python3 -m http.server 3000` | Only runtime available; config in `.claude/launch.json` |

---

## Data model (`localStorage` key: `ct_v1`)

```json
{
  "periods": [
    { "start": "YYYY-MM-DD", "end": "YYYY-MM-DD | null" }
  ],
  "settings": {
    "cycleLength": 28,
    "periodLength": 5,
    "lutealLength": 14
  }
}
```

---

## Cycle calculation logic

- **Avg cycle length** — mean of gaps between consecutive period starts (clamped 15–60d); falls back to `settings.cycleLength`
- **Avg period length** — mean of completed period durations; falls back to `settings.periodLength`
- **Ovulation day** — `nextPeriodStart − lutealLength` (default 14d)
- **Ovulation window** — ov day ±2 days
- **High libido window** — ov day −3 to ov day +1
- **PMS window** — last 5 days before next period start
- **Predictions** — generates 18 future cycles forward from last logged period start; no date limit (bug was fixed where limit anchored to wrong year)

---

## Day map (`buildMap`)

`buildMap(year)` returns a `{ [dateStr]: { cls, phase, cycleDay, flags } }` map for every day in that year.

**Priority order for `cls`** (highest → lowest):
`period-actual` → `period-predicted` → `ovulation` → `ovulation-window` → `high-libido` → `pms` → `follicular` → `luteal`

**Phase key → CSS class mapping:**
- `menstrual` phase uses `period-actual` / `period-predicted` cls
- `fertile` phase uses `ovulation` / `ovulation-window` / `high-libido` cls

---

## Calendar rendering — 3 visual modes per day

| Context | Style |
|---|---|
| **Past months** | Circle ring (`border: 1.5px solid <color>`, `border-radius: 50%`, `scale(0.8)`), no fill |
| **Current month** | Solid filled box for all days (past + future) |
| **Future months** | Transparent background + coloured dot + emoji for predicted phases |

Each day cell shows: **number** → **emoji** → **abbreviation text** (stacked vertically)

### Emoji + abbreviation map
| cls | Emoji | Abbr |
|---|---|---|
| `period-actual` | 🩸 | P |
| `period-predicted` | 🩸 | ~P |
| `ovulation` | ✨ | OV |
| `ovulation-window` | 🌸 | OW |
| `high-libido` | 🔥 | HL |
| `pms` | 🌩️ | PMS |
| `follicular` | 🌱 | FO |
| `luteal` | 🍂 | LU |

---

## UI structure

```
[Sticky Header]        — title, History + Settings buttons
[Sticky Status Bar]    — Cycle Day, Phase, Next Period, Ovulation, Avg Cycle, Avg Period
[Calendar]             — year nav, legend strip (tappable), 12-month grid
[Legend FAB]           — appears when legend scrolls out of view, opens bottom sheet
[Quick Log Bar]        — Period Start / Period End / Add Past Period (fixed bottom)
```

---

## Modal inventory

| ID | Purpose |
|---|---|
| `mAdd` | Add / edit a period entry (start + end date pickers) |
| `mHistory` | List all logged periods with delete |
| `mSettings` | Default cycle/period/luteal lengths + clear all data |
| `mDay` | Day detail — phase info, timing, blurb, gear ⚙ for edits |
| `mPhase` | Full phase info card (triggered from legend, day modal phase pill, or FAB sheet) |
| `legendSheet` | Bottom sheet legend (triggered by FAB when legend out of view) |

**z-index stack:** header/status (89–90) → legend FAB (88) → legend sheet (210) → overlays/modals (220)

---

## Day modal interaction rules

| Day type | Quick-log buttons | Gear ⚙ edit |
|---|---|---|
| Past — logged period start/end day | None | ±1/±2 day nudge control |
| Past — calculated phase (follicular, luteal, etc.) | None | None |
| Past — no phase | "Add period starting here…" | None |
| Today | Full quick-log (Period Start / Period End) | ±1/±2 if on logged period day |
| Future | None | None |

Nudge controls shift `period.start` or `period.end` by ±1 or ±2 days with validation (start can't pass end, end can't pass start).

---

## Phase info cards (`PHASE_INFO`)

Each of the 8 phases has a rich info card with: icon, name, sub-title, header gradient, `about`, `duration`, `symptoms[]`, `tip`. Triggered by tapping:
- Any legend item (inline strip or FAB sheet)
- The phase badge pill inside a day modal

---

## What's live and working
- [x] Full yearly calendar with 12-month scrollable grid
- [x] Year navigation (← →)
- [x] All 8 cycle phases calculated and colour-coded
- [x] 3 visual modes: past rings / current fills / future dots
- [x] Emoji + abbreviation indicators on every day
- [x] Sticky header + status bar
- [x] Quick-log bar (Period Start / End / Add Past)
- [x] Day detail modal with phase info, timing, blurb, tappable phase pill
- [x] ⚙ gear-gated nudge editing for period start/end dates
- [x] Floating legend FAB + bottom sheet
- [x] Phase info cards for all 8 phases
- [x] iOS PWA meta tags (add to home screen)
- [x] Open Graph tags (share sheet preview image)
- [x] App icon (Kuromi) — `icon.png`
- [x] Local timezone fix (`getFullYear/Month/Date` not `toISOString`)
- [x] Predictions bug fix (no longer anchored to wrong year)
- [x] SSH push workflow configured

---

## Possible next steps (not yet built)
- [ ] Symptom / mood logging per day (cramps, energy, mood score, notes)
- [ ] Cycle insights / stats page (cycle regularity, average lengths over time)
- [ ] Push notifications for upcoming period / ovulation
- [ ] Export data (JSON backup / restore)
- [ ] Pill/contraceptive tracking mode
- [ ] Colour theme customisation
- [ ] Cycle length shown visually on the calendar (arc or bracket)
- [ ] Onboarding flow for first-time users (enter last 2–3 periods to seed predictions)
