# Project Pulse: final handoff

Project Pulse is a static dashboard that shows each team project's name, owner, status, recent activity and priority. It was built in commit `7ee35c7` on `main`. This document explains how the agent team built it, what each file does, how to run it, what was validated, and what still needs a human in a real browser.

## Agent team and roles

| Agent | Role in this build |
| --- | --- |
| **Orchestrator** | Ran the build in phases with explicit, non-overlapping file scopes. **Phase 1:** Designer and Coder in parallel. **Phase 2:** Designer integration review. **Phase 4:** Coder final validation. Verified results and reported. The Orchestrator never implemented anything itself. |
| **Planner** | Researched the repo and wrote `docs/project-pulse-plan.md`. The plan covers the data contract, file ownership, parallel and sequential decisions, edge cases, validation expectations (section 9) and open questions (section 10). Wrote no code. |
| **Designer** | **Phase 1:** wrote `app/styles.css` in parallel with the Coder. **Phase 2:** reviewed the integrated page and fixed three issues in `app/styles.css`: the `[hidden]` grid rule, the gap left by the empty live region, and alt text for the decorative "!" marker. No `app/index.html` changes were needed. |
| **Coder** | **Phase 1:** wrote `app/project-data.json`, `app/index.html` and `.vscode/launch.json`. **Phase 4:** ran the final validation recorded below and wrote this handoff. |

## Files

| Path | Owner | Purpose |
| --- | --- | --- |
| `app/index.html` | Coder | Page markup and the inline script. The script fetches `project-data.json` with `cache: "no-store"` and renders one `.project-card` per project in a list. It sets the summary counts (total, at risk, blocked) and handles the loading, empty and error states, including `file://`. All data is inserted with `textContent`, never `innerHTML`. |
| `app/styles.css` | Designer | Visual design: the `.dashboard` layout, a responsive card grid, `.project-card` with `border-radius` and `box-shadow`, status and priority badges, focus styles and reduced-motion support. |
| `app/project-data.json` | Coder | Sample data: a `projects` array with 6 projects. Each has `name`, `owner`, `status`, `recentActivity` and `priority`, plus an optional `summary`. |
| `.vscode/launch.json` | Coder | VS Code launch configuration that serves `app/` and opens the dashboard. |

## How to run

1. Open **Run and Debug** in VS Code.
1. Select **"Run Project Pulse Dashboard"** (defined in `.vscode/launch.json`) and start it.
1. It runs `python3 -m http.server 5500` with `cwd` set to `${workspaceFolder}/app`. When the server prints its ready line, VS Code opens **http://localhost:5500/index.html**: the dashboard itself, not a directory listing. In Codespaces, port forwarding happens automatically.
1. **Stop the session before running it again.** Otherwise port 5500 stays in use and the new server fails with "Address already in use".

Do not open `app/index.html` directly as a file. Browsers block `fetch()` over `file://`, so the page shows an error telling you to use the launch configuration.

## Final validation results

The Coder ran these checks during Phase 4 against the working tree at `7ee35c7`. `scripts/validate-exercise.sh` was deliberately **not** used as a gate (see plan section 9).

| # | Check | Result |
| --- | --- | --- |
| 1 | `python3 -m json.tool app/project-data.json` | ✅ Pass (exit 0) |
| 2 | `python3 -m json.tool .vscode/launch.json` (strict JSON, no comments) | ✅ Pass (exit 0) |
| 3 | `projects` is a non-empty list, and every item has `name`, `owner`, `status`, `recentActivity`, `priority` | ✅ Pass: 6 projects, no missing fields |
| 4 | Status counts | ✅ on-track 2, at-risk 1, blocked 1, completed 2 (matches the expected at-risk 1 and blocked 1) |
| 5 | Priority counts | ✅ high 2, medium 2, low 2 |
| 6 | `app/index.html` phrases: Project Pulse, styles.css, project-data.json, project-card, status, recentActivity, priority | ✅ Pass (7/7) |
| 7 | `app/styles.css` phrases: .dashboard, .project-card, border-radius, box-shadow | ✅ Pass (4/4) |
| 8 | `app/project-data.json` phrases: projects, name, owner, status, recentActivity, priority | ✅ Pass (6/6) |
| 9 | `.vscode/launch.json` phrases: Run Project Pulse Dashboard, index.html | ✅ Pass (2/2) |
| 10 | Launch `cwd` = `${workspaceFolder}/app` | ✅ Pass |
| 11 | Launch `command` = `python3 -m http.server 5500` | ✅ Pass |
| 12 | `serverReadyAction.uriFormat` = `http://localhost:%s/index.html` | ✅ Pass |
| 13 | Pattern `Serving HTTP on .* port ([0-9]+)` matches "Serving HTTP on 0.0.0.0 port 5500 (http://0.0.0.0:5500/) ..." | ✅ Pass (captures `5500`) |
| 14 | Smoke test: `GET /index.html` | ✅ 200, body contains "Project Pulse" |
| 15 | Smoke test: `GET /styles.css` | ✅ 200 |
| 16 | Smoke test: `GET /project-data.json` | ✅ 200 |
| 17 | Smoke-test server stopped afterwards | ✅ Confirmed: a later request to port 5500 was refused (curl exit 7) |
| 18 | `app/index.html` inserts data safely | ✅ 0 uses of `innerHTML`; content is set with `textContent` and `createTextNode` |

All checks were run from the command line. No browser was used, so the checks below remain open.

## Remaining items for handoff

These checks need a real browser and were **not** run:

- **Responsive layout:** check at 320px, 375px, 768px and 1280px. There should be no horizontal scroll, and the grid should reflow from 1 column to 2 to 3. The long "Customer Data Platform Migration…" name must wrap inside its card.
- **Contrast:** text contrast should be at least 4.5:1. Use the DevTools contrast checker, or run a Lighthouse accessibility audit and aim for a score of 90 or more.
- **Keyboard:** pressing Tab should show visible focus, and badges should be readable without relying on color.
- **Reduced motion:** with `prefers-reduced-motion: reduce` emulated, animations and transitions should be suppressed.
- **Error states:**
  - Opening `app/index.html` via `file://` should show the "use Run Project Pulse Dashboard" message.
  - Temporarily pointing the fetch at a bad path should show the error state.
- **Launch in VS Code:** confirm that "Run Project Pulse Dashboard" opens the dashboard, with 6 cards inside `.dashboard`.

### Open questions (from plan section 10)

1. **Status vocabulary:** keep `on-track / at-risk / blocked / completed`, or add a separate `risk` field?
2. **`recentActivity`:** keep it as free text, or make it structured (a date plus a description) so the page can sort by it?
3. **`file://` fallback:** should the page include a copy of the data inline? The current answer is no, to avoid duplicate data, and the page shows an error instead.
4. **Port conflicts:** is "stop the other server first" enough guidance when port 5500 is taken?
5. **`index.html` ownership:** the Coder owns the file and the Designer reviews it. Should the Designer write the markup instead, which would mean a sequential handoff?
