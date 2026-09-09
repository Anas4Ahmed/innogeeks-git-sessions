# New export: aSc "lesson list" XLSX

## Context (read this first)

This is a spec for a NEW export format on the **faculty-allocator** project
(the codebase is not in this repo — this file is the handoff). Two exports
already exist:

- `backend/src/export/xlsx_writer.py` → the Academic Load sheet (per-faculty
  hour totals, `GET /api/admin/export/load-sheet`)
- `backend/src/export/asc_writer.py` → aSc Timetables **XML**
  (`GET /api/admin/export/asc-xml`)

We need a **third** export: the same underlying data as `asc_writer.py`, but
shaped as an **XLSX** matching aSc's own "lesson list" import layout (the
format aSc uses when you import lessons from Excel instead of XML). This is
NOT a redesign — it's `asc_writer.py`'s data, alternate presentation.

**Goal: ship a working v1 fast.** Where a column needs data the schema
doesn't have yet (classroom pools, subject short labels), use a simple
default and a small editable lookup — do not block on getting it perfect.
Optimization/refinement is a follow-up, not this task.

---

## Target file: exact columns

| Column | Example values |
|---|---|
| `Teacher` | `Mr. Himanshu Saxena` · multi-teacher: `Mr. Vinay Kumar, Mr. Abhishek Tyagi` · unstaffed: `Without teacher` |
| `Class` | `CSIT-3A` · multi-class (cross-cohort session): `CSIT-7A,CSIT-7B,CSIT-7C` |
| `Group` | `Entire class` · repeated once per class in a multi-class row: `Entire class,Entire class,Entire class` |
| `Subject` | short display label, e.g. `ESS1`, `OS LAB`, `AI&A`, `MIN. DEG` — **not** `subjects.code` |
| `Length` | `Single` or `Double` |
| `Lessons/week` | integer count of lessons of that Length |
| `Available classrooms` | `Home classroom` · or `Home classroom, Lab G-401, Lab G-402, Lab G-403` · blank for `Without teacher`/no-room rows |
| `Cycle` | always `All weeks` for v1 |

One row = one (faculty-set, subject, class-set) lesson card — **not** one
row per raw allocation row. Rows that share teacher(s) + subject + Length
across multiple classes/sections collapse into one row with comma-joined
`Class`/`Group` (see screenshots: `CSIT-7A,CSIT-7B,CSIT-7C` for a subject
taught identically to all three sem-7 sections by the same two faculty).

---

## Where the data comes from

Do **not** re-derive coverage/allocation logic. Reuse exactly what
`asc_writer.py` already reads: the joined `allocations` + `subjects` +
`faculty` rows for the current academic year, filtered the same way
(`is_active_this_year`, pins skipped the same way, discontinued subjects
skipped the same way — copy those filters verbatim from `asc_writer.py`).

Per allocation row you already have (or can trivially join to):
- faculty name(s) — for a (subject, section) slot with >1 faculty (labs),
  all of them, comma-joined, in the order they were assigned
- `subjects.code`, `subjects.name`, `subjects.sem`, `subjects.L/T/P`,
  `sections_count`, `is_lab`/`is_blended`/`is_project` (from
  `loader.Subject` derived properties — reuse, don't reimplement)
- `section` letter (A/B/C or `-` for a department-wide pin)
- `faculty.name` (full display name, already prefix-formatted —
  reuse `src/faculty_names.py`, do not re-split)

`Class` = `CSIT-{sem}{section}` for a normal row (drop the section suffix
entirely — no `-{section}` — when it's a pin/merged/cross-cohort row; those
list every affected class instead, comma-joined).

---

## Column derivation rules (v1 defaults — good enough, not final)

### `Teacher`
Join `faculty.name` for every faculty on that (subject, section) with `, `.
If a subject/section has zero allocations (an uncovered orphan or a
`Mentor`/admin placeholder row with no real faculty backing), use the
literal string `Without teacher` — same convention as the screenshot.

### `Subject` (the short display label — THIS IS NEW, does not exist in schema)
The screenshots use labels like `ESS1`, `C E`, `Eng. Phy`, `OS LAB`, `AI&A`,
`MIN. DEG` — these are **not** `subjects.code` (which is things like
`IT401B`, `CS504`). This is a human-facing abbreviation aSc displays.

**v1 approach — do not try to derive this algorithmically:**
1. Add one new nullable column: `subjects.asc_short_label TEXT`.
2. `init_db()` picks it up automatically (schema.sql + additive migration,
   same pattern as every other schema change in this codebase — see
   `src/db.py`'s additive-migration list).
3. Fall back to `subjects.code` when `asc_short_label IS NULL`, so the
   export works immediately for every subject with zero manual setup.
4. Expose it as an editable field wherever subjects are already edited
   (`PATCH /api/admin/subjects/{id}` — add `asc_short_label` to
   `SubjectPatch`; it is NOT a solver input, so this does not conflict with
   the existing `extra="forbid"` rule protecting `L/T/P/type/sections_count`).
5. Suffix `" LAB"` when the row is the practical component of a subject that
   has both a theory and lab delivery (mirror whatever `asc_writer.py`
   already does to tell a subject's theory row from its lab row — reuse
   that flag, don't reinvent it).

Admins can then type in the real short labels for the ~40 subjects once,
post-launch, without touching code again.

### `Length` / `Lessons/week`
Derive from the subject's per-section hours, split by component:
- **Lecture (L) hours** → one row, `Length = Single`, `Lessons/week = L`
  (each lecture hour is a separate 1-period lesson).
- **Tutorial (T) hours**, if any → same as lecture, `Single`, count = T.
  (Merge into the same row as lecture if the codebase doesn't already treat
  T as a separate deliverable elsewhere — check `asc_writer.py` for how it
  currently handles T; mirror it.)
- **Practical (P) hours** → `Length = Double`, `Lessons/week = P / 2`
  (round up if odd — flag any odd-P subject found on the live seed, don't
  silently truncate).

This matches the screenshots: a 1-hour theory subject → `Single, 1`; a
4-hour practical taught in two 2-hour blocks → `Double, 2`... though most
screenshot rows show `Double, 1`, meaning **most labs here are a single
2-hour block per week** (P=2). Verify against actual subject P values when
implementing — if a subject's P doesn't cleanly halve, log it and pick the
closer integer rather than crashing the export.

### `Available classrooms`
No classroom/room data exists anywhere in this schema today. **v1: default
every row to `"Home classroom"`.**

If you want the richer per-lab room pools shown in the screenshots
(`Home classroom, Lab G-401, Lab G-402, Lab G-403`), add one more optional
lookup — do NOT model full room-scheduling:
- new table `classroom_pools(subject_id INTEGER, room TEXT)` or a single
  `subjects.classroom_pool TEXT` (comma-separated free text is fine for v1)
- left-join it in; default to `"Home classroom"` when empty
- leave populating it to the admin, same as `asc_short_label` above

Do not build room-conflict checking. This column is descriptive, not
enforced — same spirit as the accepted "merged pin not clash-guarded"
limitation already documented for the XML export.

### `Group`
Always `"Entire class"`, repeated once per comma-joined class in the row.
Do **not** attempt to split first-year lab G1/G2 groups into separate rows
— the existing codebase already treats `lab_group` as a denormalised,
non-authoritative string (see the known G1/G2 caveat in the project docs);
replicating that ambiguity into a new export format is out of scope for v1.

### `Cycle`
Always `"All weeks"`. No per-cycle scheduling exists in this system.

### Pins (fixed subjects) and cross-cohort rows
Reuse whatever `asc_writer.py` already does for pins — it already knows how
to skip/represent `fixed_faculty_id` subjects and merged pins (see
`loader.is_merged_pin`). Do not write new pin-handling logic; call the same
helper or copy its branch verbatim.

---

## Backend implementation

**New file:** `backend/src/export/asc_xlsx_writer.py`
- Function signature mirrors `asc_writer.write_asc_xml(conn)` /
  `xlsx_writer.write_academic_load(conn)` — same input (a DB connection),
  same general shape, returns bytes/BytesIO for streaming.
- Use `openpyxl` (already a dependency, per `xlsx_writer.py`) to build a
  single-sheet workbook with the 8 headers above in row 1.
- Build rows by iterating the same query/join `asc_writer.py` uses, then
  **group by (frozenset(teacher_ids), subject_id, Length)** to collapse
  multi-class rows before writing — this collapse step is the one genuinely
  new piece of logic; keep it a small pure function so it's testable in
  isolation (`group_lesson_rows(allocation_rows) -> list[LessonRow]`).

**New endpoint** in `backend/routers/admin.py`, next to the existing export
endpoints:
```python
@router.get("/export/asc-xlsx")
def export_asc_xlsx(db=Depends(get_db)):
    data = asc_xlsx_writer.write_asc_xlsx(db)
    return StreamingResponse(
        data,
        media_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        headers={"Content-Disposition": "attachment; filename=asc_lessons.xlsx"},
    )
```
Copy the exact auth/streaming pattern from the `export/load-sheet` or
`export/asc-xml` endpoint already in that file — don't hand-roll a new one.

**Schema migration:** add `asc_short_label TEXT` (and optionally
`classroom_pool TEXT`) to `subjects` in `schema.sql`, following whatever
additive-migration convention `src/db.py` already uses for past columns
(check how a recent nullable column, e.g. `asc_short_override` on
`faculty`, was added and copy that pattern exactly).

**`SubjectPatch`:** add `asc_short_label: str | None` (and
`classroom_pool` if you build it) as optional fields. These are NOT solver
inputs — fine to allow through the existing `extra="forbid"` model.

---

## Frontend implementation

**File:** `frontend/src/pages/admin/Export.jsx`
- Add one more download button next to the existing "Download aSc XML" /
  "Download Load Sheet" buttons — copy whichever one already does the
  `GET`-and-download pattern (likely via `api/client.js`'s axios instance
  with `responseType: 'blob'`), point it at
  `/api/admin/export/asc-xlsx`, save as `asc_lessons.xlsx`.
- No new page, no new state beyond what the existing download buttons
  already have (loading spinner / error toast, if that's the existing
  pattern — match it, don't add new UI conventions).

If subject-level fields (`asc_short_label`, `classroom_pool`) should be
editable from the UI, add them to whatever form already edits a subject
(`pages/admin/Subjects.jsx` or wherever `PATCH /api/admin/subjects/{id}`
is currently called) as two plain text inputs. This can ship as a fast
follow — it is NOT required for the export itself to work, since both
columns have safe fallback defaults.

---

## Tests (keep this light — one file, a handful of cases)

`backend/tests/test_asc_xlsx_export.py`:
1. Export runs and returns a valid xlsx with the 8 headers in row 1.
2. A subject with no `asc_short_label` falls back to `subjects.code`.
3. A subject with both L and P hours produces two rows (Single + Double)
   with the right `Lessons/week` counts, on a small synthetic fixture —
   don't run this against the real solver.
4. Multi-faculty lab row collapses to one comma-joined `Teacher` string.
5. A cross-cohort/pin row (reuse whatever fixture `test_admin.py` or
   `test_solver.py` already has for IT501/BCS755P/CS590P) produces the
   comma-joined `Class`/`Group` shape instead of exploding into N rows.

Do not attempt to byte-match the screenshot's real 34-faculty data —
that's a UAT/manual check against the live seed, not a unit test.

---

## Explicitly deferred (do NOT do these now)

- Real classroom-conflict scheduling.
- Splitting first-year labs into true G1/G2 rows.
- Making `asc_short_label` mandatory or auto-derived — manual admin entry
  is the v1 answer.
- Any change to the existing `asc_writer.py` XML export — this is additive,
  the XML path is untouched.
- Handling T (tutorial) hours as anything other than merged-with-lecture,
  unless you find `asc_writer.py` already treats T specially, in which case
  mirror that instead of inventing a new rule.
