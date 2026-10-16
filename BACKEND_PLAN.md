# Backend Plan: Workout Spreadsheet Parser

## Overview
A Python + FastAPI backend that parses the `PD-SPRING26.xlsx` training plan with openpyxl and serves workouts via REST to the Expo app. MVP uses the static sample spreadsheet; later it will read directly from Google Sheets.

## Spreadsheet Analysis

- 9 sheets, one per training cycle: `Spring 26`, `Fall25`, `SoS25`, `Fall24`, `Spring24-`, `SoS24`, `Fall23`, `Spring 25`, `Copy of Fall25`.
- One row per week. Column A is the week's start date (Monday). Other columns are days of the week, with `LOCATION` columns beside Tuesday and Saturday.
- Workout text is free-form running shorthand: `MP` (marathon pace), `HM`, `10k`/`5k`/`3k`, `K` (kilometer), `m` (miles or minutes depending on context), `j` (jog), `rec` (recovery).
- Race markers appear in the Sunday column (e.g. "Chicago Marathon", "Houston").

### Known quirks the parser must handle
- **Layouts differ by sheet.** `Spring 26` has a header row, a `Thurs` column and a `GN` location column; the older sheets have a blank row 2 and slightly different columns. Detect columns by header name, not fixed index.
- **Inconsistent dates.** `Spring 26` has January/February dates stamped 2025 instead of 2026, and `Copy of Fall25` has 2026 dates mixed with 2025 ones. Infer the year from the surrounding week sequence rather than trusting the cell value.
- **Location values are messy.** Examples: `GN`, `Great Northern`, `Greath Northern`, `Mueller `, `Waller St (Martin Middle School)`. Normalize with a lookup table and trim whitespace.
- **Stray content.** Some cells hold URLs (Strava routes), `OFF`, `Recovery`, or notes like `Thu: 7-8 x 1m...` inside the wrong column. An extra "Lower Volume Version" column exists in some sheets.
- **Empty weeks.** `Fall23` begins with several weeks of blank rows.

## Data Model

```python
Workout:
  date: date
  day: str                 # "Tuesday"
  cycle: str               # "Spring 26"
  description: str         # raw text, kept as-is for MVP
  location: str | None     # normalized
  race: str | None
  alt_description: str | None   # lower-volume version, when present

Week:
  week_number: int
  start_date: date
  workouts: list[Workout]

TrainingCycle:
  name: str
  start_date: date
  end_date: date
  weeks: list[Week]
```

## Proposed Structure
```
backend/
├── main.py                # FastAPI app
├── parser/excel_parser.py # openpyxl logic, header detection, normalization
├── models.py              # Pydantic models
├── routes.py              # API endpoints
├── data/PD-SPRING26.xlsx
└── requirements.txt
```
Use a virtualenv (system Python is externally managed on this machine).

## API Endpoints
- `GET /cycles` - list cycles with name, date range, week count
- `GET /cycles/{cycle_name}` - full cycle
- `GET /cycles/{cycle_name}/weeks/{week_num}` - single week
- `GET /workouts?date=YYYY-MM-DD` or `?from=...&to=...` - query by date
- `POST /upload` - future: ingest a new spreadsheet

## Phases
1. **Parser + API (MVP):** parse all sheets into models, serve JSON, in-memory only.
2. **App integration:** Expo app fetches from the API; views by cycle, week, and today's workout; filter by location.
3. **Persistence + Google Sheets:** swap the xlsx source for the Google Sheets API, add caching/DB (SQLite or Postgres), then auth if needed.

## Key Decisions
- **No database for MVP.** Parse once at startup and serve from memory; add persistence when caching or user data requires it.
- **Keep workout text raw.** Structured parsing of intervals and paces (e.g. `8-10 x K @ HM` into reps/distance/pace) is a later enhancement, and the free-form text makes it a hard problem on its own.
- **Header-driven column mapping** so one parser handles all sheet layouts and the future Google Sheets source.

## Open Questions
- Should the app expose only the current cycle (`Spring 26`) or all historical cycles?
- How should the "lower volume version" column be surfaced in the app?
- Where will the backend be hosted (local only, Fly/Render/etc.)?

## MVP Success Criteria
- Parse every sheet without errors, including dates with the wrong year.
- All workouts retrievable via the API with normalized locations.
- Expo app can fetch and display this week's workout.
