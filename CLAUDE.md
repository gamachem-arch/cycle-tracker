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

**Dev access from iPhone:** run server then open `http://<mac-local-ip>:3000` on same Wi-Fi. Get IP with `ipconfig getifaddr en0`.

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
- **Predictions** — generates 18 future cycles forward from last logged period start; no date limit

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
[Sticky Status Bar]    — 3×2 grid: Cycle Day / Phase / Next Period / Ovulation / Avg Cycle / Avg Period
[Calendar]             — optional year nav, collapsible past/future sections, current month always expanded
[Legend FAB]           — always visible (fixed bottom-right), opens bottom sheet
[Quick Log Bar]        — floating pill above bottom edge: Period Start / End Period Today / Add Past Period
```

---

## Status bar layout

6 items in a **3-column × 2-row CSS grid** (no horizontal scroll):

| Col 1 | Col 2 | Col 3 |
|---|---|---|
| Cycle Day | Phase (emoji + pill badge) | Next Period |
| Ovulation | Avg Cycle | Avg Period |

Phase cell renders with `innerHTML`: emoji from `EMOJI[todayInfo.cls]` + `<span class="phase-badge phase-{phase}">` pill.

---

## Calendar month sections (current year only)

When viewing the current year, months are split into 3 groups:
- **Past months** (`↑ Jan – Apr`) — collapsed by default, `id="section-past"`
- **Current month** — always visible, rendered directly in the grid
- **Future months** (`↓ Jun – Dec`) — collapsed by default, `id="section-future"`

`makeSectionEl(label, count, type)` builds the collapsible wrapper. `toggleSection(id)` toggles `.collapsed` class. When not current year, all 12 months render flat with no sections.

---

## Year navigation

Hidden by default (`id="yearRow"`, starts with `.hidden`). `syncYearNav()` — called from `render()` — shows the nav only if any `db.periods` entry has a start year ≠ current year. Snaps `currentYear` back to current year if nav becomes hidden.

---

## Modal inventory

| ID | Purpose |
|---|---|
| `mAdd` | Add a period entry — inline range calendar picker (tap start → tap end) |
| `mHistory` | List all logged periods with delete |
| `mSettings` | Default cycle/period/luteal lengths + clear all data |
| `mDay` | Day detail — phase info, timing, blurb, gear ⚙ for edits |
| `mPhase` | Full phase info card (triggered from FAB sheet or day modal phase pill) |
| `legendSheet` | Bottom sheet legend (triggered by Legend FAB) |

**z-index stack:** header/status (89–90) → legend FAB (88) → legend sheet (210) → overlays/modals (220)

---

## Quick log bar

Floating pill, `bottom: 28px`, `left/right: 14px`, `border-radius: 18px`, `box-shadow`.

| Button | Class | Behaviour |
|---|---|---|
| Period Start | `btn-rose` | Shown when no active period |
| End Period Today | `btn-period-end` (solid rose, white text) | Shown when period active; asks confirmation before logging end |
| + Add Past Period | `btn-ghost` | Always visible; opens `mAdd` modal |

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
- Any item in the Legend FAB bottom sheet
- The phase badge pill inside a day modal

---

## Today indicator

`.day.today` gets `box-shadow: 0 0 0 2px var(--purple-light), 0 0 10px rgba(167,139,250,.35)` — a glowing purple ring. Works on both filled (current month) and circle (past month) day styles. No dot pseudo-element.

---

## What's live and working
- [x] Full yearly calendar with 12-month scrollable grid
- [x] Collapsible past/future month sections (current year only)
- [x] Year navigation — hidden unless periods span multiple years
- [x] All 8 cycle phases calculated and colour-coded
- [x] 3 visual modes: past rings / current fills / future dots
- [x] Emoji + abbreviation indicators on every day
- [x] Sticky header + status bar (3×2 grid, no scroll)
- [x] Phase cell in status bar shows emoji + coloured pill badge
- [x] Today indicator: glowing purple ring
- [x] Quick-log floating bar (Period Start / End Period Today / Add Past)
- [x] Period End requires confirmation dialog
- [x] Day detail modal with phase info, timing, blurb, tappable phase pill
- [x] ⚙ gear-gated nudge editing for period start/end dates
- [x] Legend FAB always visible + bottom sheet
- [x] Phase info cards for all 8 phases
- [x] iOS PWA meta tags (add to home screen)
- [x] Open Graph tags (share sheet preview image)
- [x] App icon — `icon.png`
- [x] Local timezone fix (`getFullYear/Month/Date` not `toISOString`)
- [x] SSH push workflow configured
- [x] Add Period uses inline range calendar picker (tap start → tap end, month nav, range highlight, day count)

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
