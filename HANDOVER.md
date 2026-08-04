# F&D Calculator — Handover

Single-file HTML/CSS/JS app (`index.html`, no build step, no dependencies) that
calculates Flight Duty Period (FDP) and rest requirements under **SACAA Part 127,
dual-crew (two-pilot) helicopter operations**. Hosted on GitHub Pages.

## What it does

Two inputs — **start of duty** and (optionally) **end of duty** — produce:

1. **(12h) Duty end** — theoretical max duty end if no break is taken, shown only
   when no actual end of duty has been entered. Hidden once an end is entered
   (at that point it's no longer useful — you already know when duty ended).
2. **Earliest next duty start** — the legally-minimal next duty start time.
3. **Split duty table** — either predictive (break length → resulting duty end,
   when no end of duty is entered) or concrete (start duty 2 → break → duty end,
   with non-viable rows struck through, once an end of duty is entered).
4. **Details** (collapsed `<details>`) — max flight time, local night detection,
   the rest-period explanation, the timeline gauge, and all regulatory caveats.

## Regulatory logic implemented

- **Table 1** — two-pilot max FDP is a flat 12h regardless of local start time band
  (the single-pilot column and the flight-time columns are not used/tracked).
- **7.2 split duty extension** — FDP extension = break ÷ 2, only for breaks in the
  3–10h band (below 3h = nil extension; above 10h is flagged as "not really split
  duty, that's a full rest period" rather than extrapolated).
- **8(2)(a)(i)/(ii)** — 9h rest if it includes a local night, else 10h. The app
  computes the **true legal minimum**, not just a fixed 9-or-10 branch: it scans
  from duty-end+9h to duty-end+10h for the earliest moment 8h of night coverage
  (SACAA Part 127.1's definition) is achieved, since "at least nine hours
  including a local night" can beat a flat 10h by up to 59 minutes in some cases
  (e.g. duty ending 20:30 → 9:30 rest → 06:00, not 06:30 under a naive branch).
- **"Local night" definition** — SACAA Part 127.1 (inserted by SA-CATS 2/2025,
  w.e.f. 20 June 2025): a period of 8 hours falling between 22:00 and 08:00 local
  time. This is a **confirmed SACAA definition**, not an EASA-borrowed assumption
  (an earlier version of this app mistakenly treated it as undefined — checked
  and corrected against the actual gazetted text).
- **8(2)(a)(iii)** — duty exceeding 11h needs additional rest "per the operator's
  scheme" (no number given in the regulation). **App convention: flat 11:00
  rest** applied whenever duty > 11h, since the user has no operator scheme and
  a hard "it's not defined" dead-end wasn't useful. This is clearly labelled in
  the UI as an app convention, not a SACAA figure.
- **7.2 bed requirement** — breaks ≥ 6h are marked with a 🛏 in the split table.
  Note: the regulation's actual line is "more than six hours" (i.e. > 6h, not
  ≥ 6h) — marking from 6:00 is a deliberate, disclosed over-caution by the user,
  not a misreading of the text.
- **Max flight time readout** (in Details) — Table 1's flight-time cap (8h for
  06:00–17:59 starts, 7h for 18:00–05:59 starts). Informational only; the app
  does not track actual flight time, only duty period.

## Deliberately out of scope

- **8(2)(b)** — the two-consecutive-duty-periods aggregate/intervening-rest rule.
  Both the duty-time and flight-time triggers. Was implemented once with a full
  look-back panel, then removed at the user's request as too complex for the
  simplicity goal — track manually.
- **8(2)(c)** — 60-hour cumulative duty → 24h rest rule. Needs duty history
  across many periods; deliberately kept out to avoid turning this into a
  logbook app.
- **8(2)(d)** — 18-hour duty → 18h rest rule. **Mathematically unreachable**
  under this app's own model: Table 1 max FDP (12h) + max split-duty extension
  (5h, from a 10h break) = 17h ceiling. Confirmed, not just assumed.

## Key implementation details / gotchas

- **`datetime-local` inputs with `appearance: none`** — iOS Safari's native
  datetime-local control has a minimum intrinsic width that ignores CSS `width`
  unless native appearance is stripped. This was the actual fix for an overflow
  bug that persisted through several other attempted fixes (box-sizing,
  min-width, splitting into separate date+time inputs). If you ever see
  overflow again on iOS, this is the first thing to check.
- **`step="600"`** (10-minute intervals) is set on both time inputs, but iOS's
  wheel picker has historically ignored `step` and shown every minute anyway —
  Android/desktop respect it. The "now" default value is rounded to the
  nearest 10 minutes regardless, so the convention holds even where the picker
  itself doesn't enforce it.
- **Color convention**: amber = duty end values, white = next-duty-start values,
  teal = break/rest durations. Applied consistently across the result panel,
  the split table, and the gauge.
- **`nightOverlapHours(start, end)`** is the core night-detection primitive —
  sums overlap between an arbitrary time window and the recurring 22:00–08:00
  corridor across as many days as needed. Reused both for the 9h-rest check and
  (previously) an 8(2)(b) look-back panel that's since been removed.
- The app was **audited repeatedly** by extracting its actual `<script>` block
  and driving it through a stubbed DOM in Node, then checking outputs against
  independently-written reference implementations (not just re-reading the same
  code). Worth doing this again after any non-trivial logic change — it caught
  a real bug (the naive 9-vs-10h branch missing the "at least" in the
  regulation's wording).

## UI/UX decisions worth knowing

- Went through several rounds of simplification after user testing ("too many
  fields, looks complicated"): mode toggle (normal/split) removed entirely —
  the app now always shows both the no-break and split-duty answers side by
  side, distinguished by "No rest taken" / "Rest taken — split duty" headings
  that only appear in predictive mode (no end of duty entered).
- Explanatory/regulatory text is folded into a collapsed **Details** section
  below the split table — day-to-day use is just: two inputs, two big time
  readouts, one small table.
- A "What if I enter a different rest length" feature (both a manual override
  input and a 9/10/11/12h comparison table) was built, tested, then explicitly
  reverted at the user's request — don't re-add without being asked.

## Files

- `index.html` — the app (GitHub Pages requires this exact filename at the
  repo root, or in the folder configured as the Pages source, to auto-serve).
- `F&D-Calculator-512.png`, `-192.png`, `-32.png`, `-16.png` — icon set.
- `F&D-Calculator-180.png` — apple-touch-icon.
- `F&D-Calculator-favicon.ico` — multi-resolution favicon.
  All icon files are already linked from `index.html`'s `<head>` by relative
  path — they just need to sit in the same folder when pushed to GitHub.
