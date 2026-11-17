# Project Pulse implementation plan

> The Planner wrote this plan for the Orchestrator. Agents do not stage, commit, or push. The learner controls all git operations through Copilot CLI prompts.

## 1. Summary

Project Pulse is a small static dashboard for Mona's team. There is no build step and no framework. It shows one card per project with:

- the project name
- the owner
- a status badge
- recent activity
- clear priority treatment
- an optional short summary for contributors

Each file has exactly one owner. Visual design and accessibility belong to the Designer. Data, rendering logic and the runnable launch setup belong to the Coder.

The two agents can only work in parallel if they agree on names first. The **shared contract** in section 6 fixes the CSS class names and the data fields up front.

The dashboard runs from the VS Code launch configuration **Run Project Pulse Dashboard**. That configuration serves `app/` with `python3 -m http.server 5500` and opens `http://localhost:%s/index.html`, not a directory listing.

**What the research found:**
- `app/` exists but is empty, and `.vscode/` contains only `tasks.json`.
- The Step 3 workflow (`.github/workflows/3-step.yml`) checks only that files exist and that certain phrases appear in them. Those phrases are listed in section 9.
- The devcontainer uses the `universal:2` image, so `python3` is available. The Python VS Code extension is **not** listed in the devcontainer, so the launch config must not rely on it.

## 2. File assignments

| File | Owner agent | Purpose |
| --- | --- | --- |
| `app/index.html` | **Coder** | Page structure with the exact title "Project Pulse". Links `styles.css`. An inline `<script>` loads `project-data.json` and renders one `.project-card` per project inside `.dashboard`. Also shows loading, empty and error states. Uses only the class names from the contract. |
| `app/styles.css` | **Designer** | All visual design: `.dashboard` layout, `.project-card` styling with `border-radius` and `box-shadow`, status and priority badges, typography, spacing, a responsive grid, focus styles and reduced-motion handling. |
| `app/project-data.json` | **Coder** | Strict JSON with a top-level `projects` array. Each project has `name`, `owner`, `status`, `recentActivity` and `priority`, plus an optional `summary`. Provide 5–6 realistic projects that cover every status and priority value. |
| `.vscode/launch.json` | **Coder** | Strict JSON with no comments. One configuration named "Run Project Pulse Dashboard": `cwd` is `${workspaceFolder}/app`, it runs `python3 -m http.server 5500`, and `serverReadyAction` opens `http://localhost:%s/index.html`. |

No other files are in scope. `docs/final-handoff.md` belongs to Step 4, and `.vscode/tasks.json` must not be modified.

## 3. Designer responsibilities

- **Owns `app/styles.css` only.** The Designer does not edit `index.html` or the JSON files. Markup or accessibility changes go back to the Orchestrator, who routes them to the Coder.
- **Make it look like a dashboard, not a bare page.** That means:
  - a header band with the "Project Pulse" title and a subtitle
  - a summary strip of counts
  - a card grid
- **Required selectors:** `.dashboard` and `.project-card`. Cards use `border-radius`, `box-shadow`, a clear type scale and generous padding.
- **Status badges** (`.status-badge` plus a modifier class) use a distinct color for each status. The text label is always visible, so status never depends on color alone. Text must reach at least 4.5:1 contrast against the badge background.
- **Priority treatment:**
  - `.priority-badge` plus a modifier class.
  - High-priority cards get a stronger accent, for example a left border on `.project-card--priority-high`.
  - Low priority is visually quieter but still readable.
- **Responsive layout:**
  - Single column at about 375px wide, growing to multiple columns, for example with `grid-template-columns: repeat(auto-fill, minmax(min(100%, 18rem), 1fr))`.
  - No horizontal scrolling at 320px.
- **Accessibility styling:**
  - visible `:focus-visible` outlines
  - `@media (prefers-reduced-motion: reduce)` disables transitions
  - long words wrap with `overflow-wrap: anywhere`
  - base font size of at least 16px and line-height of about 1.5
- **Style the page states:** `.dashboard-message`, `.dashboard-message--loading`, `.dashboard-message--empty` and `.dashboard-message--error`.
- **Phase 2:** review the page running in the browser and report any accessibility or hierarchy fixes needed in `index.html` to the Orchestrator. Do not edit that file.

## 4. Coder responsibilities

- **Owns** `app/index.html`, `app/project-data.json` and `.vscode/launch.json`.

**`app/index.html`**
- Head and title:
  - `<html lang="en">`
  - `<meta charset>` and a viewport meta tag
  - `<title>Project Pulse</title>`
  - `<link rel="stylesheet" href="styles.css">`
- Page structure:
  - `<main class="dashboard">` with one `<h1>` that reads "Project Pulse"
  - a list (`ul.project-grid`) of `li > article.project-card`
- Rendering:
  - Use `fetch("project-data.json")` with a relative path.
  - Build every element with `textContent`, never `innerHTML` with data values, so data cannot inject markup.
  - Show `status`, `recentActivity` and `priority` on every card, with visible labels.
- Normalization: lowercase and trim `status` and `priority`, and replace spaces with `-`.
  - Unknown values get the `--unknown` modifier but still show the raw text.
- A status region with `role="status"` and `aria-live="polite"` announces loading, empty and error states.
- Keep the script inline, because no separate `.js` file is assigned.

**`app/project-data.json`**
- Strict JSON: no comments and no trailing commas.
- Use the schema in section 6.
- Cover every status value and every priority value. Include at least one long project name to test wrapping.

**`.vscode/launch.json`** (field specification)

| Field | Value |
| --- | --- |
| `version` | `"0.2.0"` |
| `configurations[0].name` | `"Run Project Pulse Dashboard"` |
| `type` / `request` | `"node-terminal"` / `"launch"`. This debugger is built into VS Code, so no Python extension is needed. |
| `command` | `"python3 -m http.server 5500"` |
| `cwd` | `"${workspaceFolder}/app"` |
| `serverReadyAction` | `{ "pattern": "Serving HTTP on .* port ([0-9]+)", "uriFormat": "http://localhost:%s/index.html", "action": "openExternally" }` |

- Validate every file you own before reporting (see section 9).

## 5. Ordered implementation steps (phases)

| Phase | Work | Owner | Files | Mode |
| --- | --- | --- | --- | --- |
| 0 | Orchestrator confirms the shared contract in section 6 and gives each agent its file scope. | Orchestrator | none (plan only) | Sequential gate |
| 1a | Write the stylesheet against the contract. | Designer | `app/styles.css` | **Parallel** with 1b |
| 1b | Write the data file, then `index.html` rendering against the contract, then `launch.json`. | Coder | `app/project-data.json`, `app/index.html`, `.vscode/launch.json` | **Parallel** with 1a |
| 2 | Integration review: run the page in a browser, and Designer reports any markup or accessibility issues. | Designer reviews; fixes routed to file owners | Fixes only in the owner's files | Sequential, after 1a and 1b |
| 3 | Apply fixes from Phase 2. Only the owner of each file edits it. | Coder and/or Designer | Their own files only | Parallel if fixes touch different files |
| 4 | Final validation (section 9) and report to the learner. | Orchestrator, with checks run by the Coder | none | Sequential |

## 6. Dependencies and shared contract

**How the files depend on each other:**
- `index.html` depends on the `project-data.json` **schema**, which is fixed below, not on the finished file.
- `styles.css` depends on the **class names** that `index.html` produces, also fixed below.
- `launch.json` depends only on the path `app/index.html` and port 5500.
- Phase 2 depends on all of Phase 1 being finished.

**Data contract (`app/project-data.json`):**

| Field | Type | Required | Allowed values / notes |
| --- | --- | --- | --- |
| `projects` | array (top level) | yes | Can be empty. |
| `name` | string | yes | Can be long; must wrap. |
| `owner` | string | yes | Person or team. |
| `status` | string | yes | `on-track`, `at-risk`, `blocked`, `completed`. Anything else is treated as unknown. |
| `recentActivity` | string | yes | Short sentence, e.g. "Merged PR #42 on 2026-11-12". |
| `priority` | string | yes | `high`, `medium`, `low`. Anything else is treated as unknown. |
| `summary` | string | no | One or two sentences for contributors. |

**CSS class contract (the HTML produces these, the CSS styles them):**

| Class | Element / meaning |
| --- | --- |
| `.dashboard` | `<main>` wrapper |
| `.dashboard-header`, `.dashboard-title`, `.dashboard-subtitle` | Header band, the `<h1>` "Project Pulse", and the tagline |
| `.dashboard-summary`, `.summary-item` | Count strip (total projects, at-risk, blocked) |
| `.project-grid` | `<ul>` that holds the cards |
| `.project-card` | `<article>` for one project, plus a modifier `.project-card--priority-{high\|medium\|low\|unknown}` |
| `.project-name` | `<h2>` inside the card |
| `.project-owner`, `.project-summary`, `.project-activity` | Card fields. The activity field has a visible "Recent activity" label. |
| `.project-meta` | Wrapper for the badges |
| `.status-badge` plus `.status-badge--{on-track\|at-risk\|blocked\|completed\|unknown}` | Status pill with a text label |
| `.priority-badge` plus `.priority-badge--{high\|medium\|low\|unknown}` | Priority pill with the text "Priority: High" and so on |
| `.dashboard-message` plus `--loading`, `--empty`, `--error` | State messages inside the live region |

Changing either contract after Phase 0 needs Orchestrator approval and must be shared with both agents.

## 7. Parallel work decisions

- **Phase 1a (Designer) and Phase 1b (Coder) run in parallel.** This is safe because:
  - The file scopes do not overlap: the Designer has only `styles.css`, and the Coder has the other three files.
  - Both sides build against the fixed contract in section 6, so neither needs the other's finished output.
- **Within the Coder's work, these steps are sequential:** `project-data.json` first, then `index.html`. Writing real sample data first makes it easier to test rendering and edge cases. `launch.json` is independent and can be done at any point.
- **Phase 2 is sequential.** A visual and accessibility review needs the integrated page running in a browser.
- **Phase 3 fixes may run in parallel** only if they touch different files. Fixes to the same file go to its single owner, one after another.
- **Phase 4 validation is last.** It checks the integrated result.

## 8. Edge cases

- **Opened via `file://`:** most browsers block `fetch()` for local files. Show an error message that names the cause and tells the user to use "Run Project Pulse Dashboard". Do not leave a blank page.
- **Fetch or HTTP failure** (missing file, or a response with `!response.ok`): show the error state and log details to the console.
- **Malformed JSON:** catch the parse error and show the error state.
- **Missing `projects` key, or a value that is not an array:** show the error state saying the data is invalid.
- **Empty `projects` array:** show the empty state ("No projects yet"). The summary counts show 0.
- **Project missing a required field:** render the card anyway, with the placeholder "Not provided" for that field. One bad record must not break the whole page.
- **Unknown or differently cased status or priority** (e.g. "At Risk"): normalize the value. If it is still unknown, use the `--unknown` modifier and show the raw text.
- **Long names or activity text:** wrap them; never overflow the card.
- **HTML in data values:** `textContent` keeps it from being interpreted as markup.
- **Port 5500 already in use:** the server fails with "Address already in use". The learner must stop the earlier preview first.
- **Stale cache after editing the data:** fetch with `{ cache: "no-store" }`.
- **Codespaces:** `localhost` forwarding is handled automatically. The URL must keep `/index.html` so it never shows a directory listing.
- **Accessibility:**
  - one `<h1>`, then `<h2>` per card
  - list semantics for the cards
  - status never shown by color alone
  - 4.5:1 text contrast
  - visible keyboard focus
  - reduced-motion support
  - `lang` attribute set on the page

## 9. Validation expectations

**JSON validity**
- `python3 -m json.tool app/project-data.json` exits 0.
- `python3 -m json.tool .vscode/launch.json` exits 0. This also proves the file has no comments.
- A small check confirms `projects` is a non-empty list and that every item has `name`, `owner`, `status`, `recentActivity` and `priority`.

**Phrases the Step 3 workflow checks for** (case-insensitive)

| File | Must contain |
| --- | --- |
| `app/index.html` | `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority` |
| `app/styles.css` | `.dashboard`, `.project-card`, `border-radius`, `box-shadow` |
| `app/project-data.json` | `projects`, `name`, `owner`, `status`, `recentActivity`, `priority` |
| `.vscode/launch.json` | `Run Project Pulse Dashboard`, `index.html` |

**Launch config**
- `cwd` is `${workspaceFolder}/app`.
- `command` is `python3 -m http.server 5500`.
- `serverReadyAction.uriFormat` is `http://localhost:%s/index.html`.
- The pattern matches the server's startup line, "Serving HTTP on 0.0.0.0 port 5500 …".

**Runtime smoke test** (Coder runs it, then stops the server)
- Run `cd app && python3 -m http.server 5500`.
- `curl -s -o /dev/null -w "%{http_code}" http://localhost:5500/index.html` returns 200, and the body contains "Project Pulse".
- `http://localhost:5500/project-data.json` returns 200.
- In VS Code, selecting **Run Project Pulse Dashboard** opens the dashboard itself, not a directory listing. One `.project-card` appears for each project inside `.dashboard`.

**Responsive and accessibility checks**
- At 320–375px, 768px and 1280px: no horizontal scroll, and the grid reflows from 1 column to 2 to 3 or more.
- Pressing Tab shows visible focus, and badge text is readable without color.
- Contrast is at least 4.5:1, e.g. checked with the browser DevTools contrast checker or Lighthouse accessibility audit; aim for a score of 90 or more.

**State checks**
- Temporarily point the fetch at a bad path: the error state appears.
- Opening the file directly via `file://` shows the error message telling the user to use the launch configuration.

**Do not use `scripts/validate-exercise.sh` as a pass/fail gate here.** It checks that the template itself is intact and deliberately fails once learner files such as `app/*` are tracked by git.

## 10. Open questions

1. **Status vocabulary:** is `on-track / at-risk / blocked / completed` acceptable, or should "priority or risk" be a separate `risk` field? This plan uses priority only and leaves `risk` out of scope.
2. **`recentActivity` format:** should it stay free text, or become structured data (e.g. a date plus a description) so the page can sort by it? This plan uses free text and no sorting; the order follows the JSON file.
3. **Fallback for `file://`:** should `index.html` include a copy of the data inline so the page works without the server? This plan says no, to avoid two copies of the data drifting apart, and shows a helpful error instead.
4. **Port:** this plan fixes the port at 5500, as Step 3 requires. Should the plan also cover what to do if that port is already taken, or is "stop the other server first" enough?
5. **Ownership of `index.html` markup:** the Coder owns the file, and the Designer reviews it in Phase 2. If the Orchestrator would rather the Designer write the static markup, `index.html` would need a sequential handoff (Designer, then Coder) instead of running in parallel.
