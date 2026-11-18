# Project Pulse Dashboard: Implementation Plan

> This document is ready to save as-is to `docs/project-pulse-plan.md`. One thing to know first: the Orchestrator and Planner don't have edit tools, so the Orchestrator has to send a Coder (or Designer) task to write this file. `docs/project-pulse-plan.md` should be assigned only to that one agent.

---

## 1. Summary

Project Pulse is a small static dashboard that shows Mona's team projects as cards. Each card shows the project's **name, owner, status, recent activity, and priority**, plus a short summary for contributors. Four files are delivered:

| File | Owner | Purpose |
|---|---|---|
| `app/project-data.json` | **Coder** | Data model and sample data: a top-level `projects` array |
| `app/index.html` | **Coder** (Designer reviews) | Page structure, plus inline JS that loads the data and renders the cards |
| `app/styles.css` | **Designer** | Visual design, status and priority badges, responsive layout, accessibility styling |
| `.vscode/launch.json` | **Coder** | The **Run Project Pulse Dashboard** launch config: serves `app/` on port 5500 and opens `index.html` |

What I found in the repo:
- `app/` is empty.
- `.vscode/` contains only `tasks.json`. That file runs on folder open and **must not be modified**.
- Nothing in the repo uses a build tool or framework, and there's no `package.json`. The dashboard should be plain HTML, CSS and vanilla JS with no dependencies.
- The page uses `fetch('project-data.json')`, so it **must be served over HTTP**. Opening it from `file://` fails, which is why the launch config is needed.
- The Step 3 grader (`.github/workflows/3-step.yml`) only checks for files and literal text. These strings must appear:
  - in `index.html`: `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority`
  - in `styles.css`: `.dashboard`, `.project-card`, `border-radius`, `box-shadow`
  - in the data file: `projects`, `name`, `owner`, `status`, `recentActivity`, `priority`, and the file must parse as JSON
  - in `launch.json`: `Run Project Pulse Dashboard`, `index.html`, and the file must parse as JSON

The way to run the **Designer** and **Coder** in parallel is to **agree on a shared contract first** (Step 0 below). Once the class names, DOM structure and data schema are fixed, `styles.css` and `index.html` can be written at the same time without either one waiting on the other.

---

## 2. Shared contract (Step 0, fixed before any parallel work)

Both specialists build against this contract. Any change to it has to go through the Orchestrator.

### 2a. Data schema (`app/project-data.json`)
```
{
  "projects": [
    {
      "id": "string (kebab-case, unique)",
      "name": "string",
      "owner": "string",
      "status": "On Track" | "At Risk" | "Blocked" | "Completed",
      "recentActivity": "string (short sentence)",
      "lastUpdated": "YYYY-MM-DD (optional)",
      "priority": "High" | "Medium" | "Low",
      "summary": "string (1–2 contributor-friendly sentences, optional)"
    }
  ]
}
```
- Fields the brief requires: `name`, `owner`, `status`, `recentActivity`, `priority`.
- Optional extras: `id`, `summary` (the brief asks for a "short contributor-friendly summary") and `lastUpdated`.
- Include 5 or 6 sample projects so that every status value and every priority value appears at least once.

### 2b. DOM structure and CSS hooks
```
body
└── div.dashboard
    ├── header.dashboard-header
    │   ├── h1.dashboard-title  → "Project Pulse"
    │   └── p.dashboard-subtitle
    ├── section.dashboard-summary (optional: counts per status) [aria-label]
    ├── p.dashboard-message [role="status" aria-live="polite"]  (loading / empty / error)
    └── main
        └── ul.project-grid
            └── li > article.project-card  [aria-labelledby → card title id]
                ├── header.project-card-header
                │   ├── h2.project-card-title          (name)
                │   └── span.status-badge.status-badge--{slug}   (status text)
                ├── p.project-card-owner               ("Owner: …")
                ├── p.project-card-summary             (summary, if present)
                ├── div.project-card-activity          (label "Recent activity" + recentActivity text)
                └── footer.project-card-footer
                    └── span.priority-badge.priority-badge--{slug}  ("Priority: High")
```
- How to build `{slug}`: lowercase the value and replace spaces with `-`. The results are `on-track`, `at-risk`, `blocked`, `completed` for status and `high`, `medium`, `low` for priority.
- Values not in the list fall back to `status-badge--unknown` or `priority-badge--unknown`.
- `.project-card--loading` is an optional modifier if the Designer wants skeleton loading cards.

---

## 3. Ordered implementation steps

### Step 0: Lock the contract (Orchestrator)
- **Files:** none. The contract above is the plan itself.
- **Action:** the Orchestrator copies sections 2a and 2b word for word into both the Designer and Coder delegation prompts.
- **Validation:** both prompts contain the same class names and field names.

### Step 1A: Data model (**Coder**)
- **File:** `app/project-data.json`
- Strict JSON with a top-level `projects` key and 5 or 6 projects that follow schema 2a.
- **Validation:** `python3 -m json.tool app/project-data.json` succeeds. Every project has all 5 required keys. Every status and priority value appears at least once.

### Step 1B: Launch configuration (**Coder**)
- **File:** `.vscode/launch.json` (new file; leave `.vscode/tasks.json` alone)
- Strict JSON, no comments. Recommended shape:
  - `"version": "0.2.0"`
  - a configuration with `"name": "Run Project Pulse Dashboard"`, `"type": "node-terminal"`, `"request": "launch"`
  - `"command": "python3 -m http.server 5500"`
  - `"cwd": "${workspaceFolder}/app"`
  - `"serverReadyAction"` containing:
    - `"pattern": "Serving HTTP on .* port ([0-9]+)"`
    - `"uriFormat": "http://localhost:%s/index.html"`
    - `"action": "openExternally"`
- Why `node-terminal`: VS Code's built-in JavaScript debugger provides it, so nothing needs installing. The devcontainer doesn't list the Python extension, so a `"type": "python"` launch would depend on an extension that may not be there.
- **Validation:**
  - `python3 -m json.tool .vscode/launch.json` succeeds.
  - The file contains `Run Project Pulse Dashboard` and `index.html`.
  - `cwd` ends in `/app`.

### Step 1C: HTML structure and rendering (**Coder**)
- **File:** `app/index.html`
- Contents:
  - `<!doctype html>`, `lang="en"`, `meta charset` and `viewport`
  - `<title>Project Pulse</title>` and an `<h1>` that says Project Pulse
  - `<link rel="stylesheet" href="styles.css">`
- Use an inline `<script>` (no extra JS file, which keeps the four-file scope). It should:
  1. Call `fetch('project-data.json')` and check `response.ok`. On failure, show an explicit error.
  2. Check that `data.projects` is an array.
  3. Build each card with `document.createElement` and `textContent`, **not `innerHTML`**, so data can't inject markup.
  4. Read `project.name`, `project.owner`, `project.status`, `project.recentActivity` and `project.priority` explicitly. This also puts the literal text the grader looks for into the file.
  5. Apply the slug modifier classes from 2b.
  6. Optionally fill in the per-status summary counts.
- **States to handle:** loading (initial message), success, empty array ("No projects yet"), and fetch or parse error ("Could not load project-data.json. Run the 'Run Project Pulse Dashboard' launch configuration.").
- **Validation:** the file contains `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity` and `priority`. The markup matches 2b. There are no console errors when served.

### Step 1D: Visual design (**Designer**)
- **File:** `app/styles.css`
- What it needs:
  - CSS custom properties for colours, spacing, radius and shadow.
  - A system font stack and readable line-height.
  - A `.dashboard` container with max-width, centred, generous padding.
  - `.project-grid` as a CSS grid using `repeat(auto-fill, minmax(280px, 1fr))` with a gap, and list styles removed.
  - `.project-card` with background, `border-radius`, `box-shadow`, padding, a clear title hierarchy, and a hover lift that is turned off under `prefers-reduced-motion`.
  - Status badges styled as pills. Colours: on-track green, at-risk amber, blocked red, completed blue or grey, unknown neutral. The **text label is always visible**, so colour is never the only signal.
  - Priority treatment: a pill, a left border accent on the card, or both. High priority should stand out most.
  - Text contrast that meets WCAG AA (4.5:1 for body text, including badge text on badge backgrounds).
  - Long names and activity text wrap with `overflow-wrap: anywhere`.
  - Visible `:focus-visible` styles.
  - Styles for `.dashboard-message`, including an error variant.
  - Breakpoints: one column below about 600px. Also tighten header spacing at small sizes.
  - Optional: `prefers-color-scheme: dark`.
- **Validation:** the file contains `.dashboard`, `.project-card`, `border-radius` and `box-shadow`. Every class in 2b has a rule. Contrast spot-checks pass.

### Step 2: Integration review (sequential: **Designer** reviews, then **Coder** applies)
- **2a. Designer** reads `app/index.html` (read-only) and the rendered page. They report any markup changes needed for hierarchy or accessibility, such as heading levels, ARIA, label wording or badge text.
- **2b. Coder** applies the agreed changes to `app/index.html`. If a class name changes, the Designer updates `app/styles.css` in a separate follow-up phase, never at the same time.
- **Validation:** every class used in the HTML and JS is styled in the CSS, and every class in the CSS is used. Grep both files for the class names to confirm.

### Step 3: Final verification (Orchestrator, with Coder running the commands)
- See section 8.

---

## 4. File assignments

| Step | Agent | Creates / modifies | Read-only access |
|---|---|---|---|
| 0 | Orchestrator | none | plan |
| 1A | Coder | `app/project-data.json` | plan |
| 1B | Coder | `.vscode/launch.json` | `.vscode/tasks.json`, `.devcontainer/devcontainer.json` |
| 1C | Coder | `app/index.html` | plan, `app/project-data.json` |
| 1D | Designer | `app/styles.css` | plan |
| 2a | Designer | none (review only) | `app/index.html`, `app/styles.css` |
| 2b | Coder | `app/index.html` | Designer feedback |
| 2c (if needed) | Designer | `app/styles.css` | `app/index.html` |
| 3 | Coder (runs checks) | none | all four files |

Files that must not be modified: `.vscode/tasks.json`, `.devcontainer/*`, `.github/*`, `scripts/*`, `docs/agent-team.md`.

---

## 5. Dependencies

- **Step 0 comes before everything.** The CSS hooks and data fields are the only link between the Designer and Coder work.
- **1C depends on 1A's schema, not on the finished file.** The field names are fixed in 2a, so the Coder can write 1A and 1C in the same phase. It's simplest to write 1A first, then 1C.
- **1C depends on 1B to be tested at runtime.** `fetch` needs the HTTP server. Static checks don't need it.
- **1D depends only on Step 0.**
- **Step 2 depends on 1C and 1D both being finished.**
- **Step 3 depends on Step 2.**
- Shared dependencies between Designer and Coder:
  - the class names and DOM structure (2b)
  - the status and priority value lists, which drive the badge modifier classes (2a)
  - the message and error-state element (`.dashboard-message`)

---

## 6. Parallel work decisions

**Phase 1 runs in parallel because the file scopes don't overlap:**
- **Coder:** `app/project-data.json`, `.vscode/launch.json`, `app/index.html` (one agent, three files, in the order 1A → 1B → 1C)
- **Designer:** `app/styles.css`

This is safe because no file has two writers and the contract removes any data dependency between the two tracks.

**These parts must run one after another:**
- Step 0 before Phase 1.
- Step 2a (Designer review) before 2b (Coder edits), then 2c (Designer CSS follow-up) only if class names changed. Each of these touches a file the other agent depends on, so they must not overlap.
- Step 3 last.
- Saving `docs/project-pulse-plan.md` comes before Phase 1. It belongs to a single agent.

---

## 7. Edge cases

1. **Opened via `file://`:** `fetch` fails, so show a clear error message telling the user to use the launch config.
2. **Port 5500 already in use:** the server exits with "Address already in use" and `serverReadyAction` never fires. Mention this in the handoff and tell the learner to stop any earlier preview first. Keep the port fixed at 5500 to match the Step 3 instructions.
3. **Directory listing instead of the dashboard:** happens if `cwd` isn't `app/` or the URL doesn't end in `index.html`. Both settings are fixed in Step 1B.
4. **Codespaces forwarding:** `openExternally` on `localhost` is forwarded automatically in Codespaces. If the browser doesn't open, the learner can use the Ports panel.
5. **Malformed JSON or missing `projects` array:** caught and shown as an error, and logged with `console.error`.
6. **Empty `projects` array:** show an empty-state message, not a blank grid.
7. **Missing optional fields or unknown status/priority:** show "—" or leave the element out, and use the `--unknown` badge class. Never render the text "undefined".
8. **Long text or many projects:** text wraps and the grid reflows. Test with a deliberately long project name.
9. **Untrusted data:** use `textContent` only, never `innerHTML`.
10. **Accessibility:** status and priority are never shown by colour alone, headings go h1 → h2, focus outlines are visible, and loading and error messages are announced through the `aria-live` region.
11. **Reduced motion:** hover transitions are turned off under `prefers-reduced-motion: reduce`.
12. **JSON with comments:** `launch.json` must be strict JSON. VS Code accepts comments, but `python3 -m json.tool` (the grader) does not.

---

## 8. Validation expectations

**After Step 1A:** `python3 -m json.tool app/project-data.json` exits 0. Each project has `name`, `owner`, `status`, `recentActivity` and `priority`. Every status and priority value appears.

**After Step 1B:** `python3 -m json.tool .vscode/launch.json` exits 0. A grep finds `Run Project Pulse Dashboard`, `index.html`, `${workspaceFolder}/app` and `http://localhost:%s/index.html`. `tasks.json` is unchanged (`git diff --stat .vscode/tasks.json` is empty).

**After Step 1C:** a grep of `app/index.html` finds `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity` and `priority`. There is no `innerHTML` used for data.

**After Step 1D:** a grep of `app/styles.css` finds `.dashboard`, `.project-card`, `border-radius` and `box-shadow`. Every 2b class has a rule.

**After Step 2:** the class names in the HTML and JS match the selectors in the CSS. The Designer approves the hierarchy and accessibility.

**Final verification (Step 3):**
1. Static checks: run all of the greps and JSON checks above. These are the same checks as `.github/workflows/3-step.yml`.
2. Runtime smoke test (Coder):
   - Run `cd app && python3 -m http.server 5500` in the background.
   - `curl -s localhost:5500/index.html` should contain "Project Pulse".
   - `curl -s localhost:5500/project-data.json` should parse.
   - Stop the server afterwards.
3. Manual check (learner):
   - Run and Debug → **Run Project Pulse Dashboard** opens `/index.html`, not a directory listing.
   - The cards render with badges.
   - Resizing to phone width shows a single column.
   - Keyboard tabbing shows focus outlines.
   - There are no console errors.
4. Error-path check (optional): temporarily rename the data file in a scratch copy, or open `index.html` from `file://`, and confirm the error message appears. Don't leave changes behind.
5. `bash scripts/validate-exercise.sh` checks the template. Its "learner files not tracked" check may fail once you commit your own files. That failure is expected for a learner and can be ignored.

---

## 9. Open questions

1. **Who writes this plan file?** The Orchestrator and Planner have no edit tool, so the recommendation is to assign `docs/project-pulse-plan.md` to the Coder.
2. **`node-terminal` versus a shell task:** I'm confident `node-terminal` together with `serverReadyAction` works in VS Code and Codespaces through the built-in JS debugger, but I haven't tested it in this Codespace. Fallback: a `preLaunchTask` with a background shell task. That would mean editing `tasks.json`, which is out of scope, so use it only if the learner approves.
3. **Summary strip:** should the per-status counts strip ship, or is it optional polish? The plan treats it as optional.
4. **Dark mode:** include it or not? The Designer decides.
5. **Sample data contents:** placeholder project names and owners are fine unless Mona's team supplies real ones.
