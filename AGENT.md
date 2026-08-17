# Agent Operating Instructions

## Recall → Actuate → Persist

This is the core loop. Follow it for every task:

**Before acting:**
1. Read EXISTING `knowledge/<project>.md` — do NOT guess state from memory
2. Find the relevant procedure in `skills/`

**Do the work.**

**After acting:**
3. Append a Timeline entry to `knowledge/<project>.timeline.md`:
   ```
   ### YYYY-MM-DDTHH:MM
   - WHAT: what changed/decided/found
   - WHY: rationale
   ```
4. Update stale sections in `knowledge/<project>.md` to reflect current state

## File Structure

- `knowledge/`  — `*.md` current state + `*.timeline.md` append-only history
- `assets/`     — code, scripts, examples (one subdir per project)
- `skills/`     — procedures (load before doing anything complex)
- `HANDOVER.md` — start here

## Rules

- Read before write — always read the knowledge file before adding facts
- Never guess state — if uncertain, read the file
- Timeline is append-only — never edit past entries
- Knowledge is current state only — no history prose in `*.md`, history lives in `*.timeline.md`
