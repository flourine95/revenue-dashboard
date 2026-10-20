# How AI was used on this project

The challenge asked how I work with AI, from receiving a requirement to trusting the code. This document describes the process I actually followed here, including the mistakes the process caught.

## The process

### 1. Read the brief before prompting

I read the task, the reference screenshots and the tech requirements, then wrote down the shape of the system myself: two applications, one database, sessions for the admin, data relative to today so the year-over-year view always has something to compare. AI was not involved in this step. Decisions like "server sessions instead of JWT", "a 364-day shift so weekdays stay aligned" and "fetch the comparison payload up front" came before any code existed.

### 2. Build in small, verifiable slices

Each slice was one commit: schema and seed, auth, dashboard endpoint, admin CRUD, the dashboard view, the insight tabs, admin screens, PNG export. For each one I gave AI the goal plus the constraints of the codebase (error shape, validation library, where environment variables are read). Small slices mean a wrong answer is visible in a small diff.

### 3. Verify with something that fails

My rule is that AI output counts as a hypothesis until a command can reject it. For every slice:

- `pnpm typecheck` and `pnpm lint` for structural mistakes.
- Unit tests for pure logic (period math, KPI rounding, the score formula, chart scaling).
- Endpoint tests against a real throwaway PostgreSQL database, not mocks, including the auth guard returning 401 and the login lockout returning 429.
- A manual pass in the browser for anything with a visual result.

### 4. Review the diff, not just the green checkmarks

Passing tests only prove the tested behaviour. Reading the diff is where I found most of the real problems, listed below.

## What AI did and what I did

| Activity | AI | Me |
| --- | --- | --- |
| Reading the brief, estimating the work | No | Yes |
| Architecture and data model decisions | Options discussed | Final call |
| Writing code inside an agreed slice | Most of it | Review every diff |
| Test suites | Drafted | Decided what deserved a test |
| Running and checking the result | No | Yes |
| Finding the problems below | Partly, when asked to review | Verified each one by hand |

AI never decided a schema change, an API contract or a UI layout on its own. Those were written down first, then implemented.

## Problems the process caught

These are concrete, and they are the reason the verification step exists.

1. **The chart axis ignored the reference design.** The scale was computed from the tallest bar, so a quiet week produced a 0–3k axis and a busy week produced 0–5k. Comparing weeks became unreliable, and it looked nothing like the reference. Fixed by pinning the reference grid (0.75k steps) and only growing it when data would clip.

2. **The legend claimed a series that was never drawn.** "Labour Costs (Previous)" had a colour, a data field and a legend entry, but no bar was rendered. A reader would trust the legend and see nothing. Found by comparing the rendered chart with the reference image, not by any test.

3. **A tooltip formatted covers as currency.** Covers would have shown as `$986`. Caught while reviewing the tooltip path.

4. **The PNG export filter could blank the whole card.** The filter skipped any element carrying the marker attribute. If that attribute ever moved to a wrapper, the export would produce an empty image. Rewritten to skip only the marked subtree.

5. **Network failures were reported as unknown errors.** `fetch` rejects rather than returning a status when the API is down, so the UI showed a vague message instead of "cannot reach the API". Now mapped to a typed 503.

6. **No limit on repeated sign-ins.** Twelve wrong passwords in a row all returned 401. Added a bounded, tested lockout.

7. **A shutdown claim in the README was stronger than the code.** The README said "graceful shutdown". On Windows, the signal never reaches the handler when the process is spawned, so the wording was narrowed to what is actually true.

Each of these is a case where the AI-written code looked correct and the tests passed. They were caught by reading, by comparing against the reference, or by deliberately trying to break the thing.

## Rules I follow

- AI output is a hypothesis. A command must be able to reject it.
- No generated dependency, migration or API contract goes in without me being able to explain it line by line.
- When AI and I disagree, the brief and the tests win, not confidence.
- Everything shipped can be run from the README on a machine with nothing installed.

## Known gaps

Documented in the README rather than hidden: the selected week and tab are not in the URL, there is no committed browser end-to-end suite, the login lockout state is in memory so it resets on restart, and only one CORS origin is supported.
