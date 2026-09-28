# Output styles

This file describes the **steps followed** to add Concept 34 (**output styles**) to the Claude Agent SDK Lab: what was
read, what was probed, what was decided, how it was tested, and what the tests changed. To learn the concept itself,
read [Tab34-Output-styles.md](Tab34-Output-styles.md) (the same text is in `readme.md`).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 34 | Output styles: style files and their frontmatter, the `outputStyle` setting, project / user / plugin styles, what reaches the API (a wire tap), what `keep-coding-instructions` removes from the `claude_code` preset, silent failures, switching styles in one session (`applyFlagSettings`, `reloadOutputStyles`, `updateSettings`), a host-side check | `/api/c34/styles`, `/who`, `/compare`, `/live`, `/code` | [Tab34-Output-styles.md](Tab34-Output-styles.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample33, so `npm install` is not needed. Open the **34. Output styles** tab.
`ANTHROPIC_API_KEY` must be in `.env`. Start it from a normal terminal, not from inside Claude Code (see Tab16). Each
button costs about $0.02, and button 3 about $0.04.

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text, as in sample28 to sample33. A grep of
the `Tab*.md` files and of `server/` and `src/` found no `outputStyle`, `toolAliases`, `permissionPromptToolName`,
`spawnClaudeCodeProcess` or `promptSuggestions`, although all of them are in `sdk.d.ts` (`0.3.281`). Four topics were
offered, and **output styles** was chosen. It follows Concept 9 (system prompts) and Concept 16 (settings).

## Step 2: Read what the course already said

sample34 held a copy of sample33 without `node_modules`, so `node_modules` was copied from sample33.

| Read | To learn |
|---|---|
| `Build-steps.md`, `32-plan-mode.ts`, `Concept32PlanMode.tsx` | The latest style: strict zod bodies, SSE events, a setups table (button 0), a folder per run, a fake `CLAUDE_CONFIG_DIR`, `#region` + `/code`, a host check |
| `29-images-files.ts` | The wire tap (`ANTHROPIC_BASE_URL` → a local proxy that forwards and records requests) |
| `14-thinking-effort-models.ts`, `26-query-control.ts` | How the lab calls `applyFlagSettings` and `initializationResult()` |
| `server/index.ts`, `src/App.tsx`, `src/lib/sse.ts`, `src/styles.css` | Mounting, the tab list, SSE, the CSS classes to reuse |

## Step 3: Check the types and the CLI

| Found | Used for |
|---|---|
| No `outputStyle` in `Options`. `Settings.outputStyle?: string` ("Controls the output style for assistant responses") | `settings: { outputStyle }` |
| `SDKSystemMessage.output_style`, `SDKControlInitializeResponse.available_output_styles` | The init row, the "listed" column |
| `Query.reloadOutputStyles()`, `Query.updateSettings('localSettings' \| 'userSettings', …)` (allowlist: `outputStyle` for localSettings), `applyFlagSettings` | Part C |
| `projectConfigRoot` lists `output-styles` among the `.claude` config trees; plugins have an `output-styles` folder | Where style files live |
| In the CLI binary: the style loader reads `name`, `description`, `keep-coding-instructions`, `force-for-plugin` ("only applies to plugin output styles. Ignoring."); plugin styles are named `<plugin>:<name>`; the preset's coding section is kept only when `keepCodingInstructions === true` | The frontmatter table, Part B |

## Step 4: Probe before designing

A scratchpad script (in a temporary `.probe/` folder) ran real sessions with Haiku 4.5 through a wire tap, each with its
own folder and `CLAUDE_CONFIG_DIR`, and dumped every request:

| Probe | Result | Decision |
|---|---|---|
| No style | `output_style: "default"`, 5 built-ins: `default, Proactive, Concise, Explanatory, Learning` | The built-in list |
| Where the text goes | Not in the system prompt: two system-reminders in the first user message (`# Output Style: <name>` + text, "X output style is active…") | The tap reads the reminders; "what the API got" |
| `claude_code` preset, style without / with `keep-coding-instructions` | Without: −3,264 chars, `# Doing tasks` removed. With: +57 chars. Both: the first line says "according to your Output Style" | Part B (Tutor vs Tutor (keeps coding)) |
| Project style with `settingSources: []` / `['project']` | Not listed, not applied / applied | Button 0, rows 4 and 5 |
| User style in `CLAUDE_CONFIG_DIR/output-styles` + `['user']` | Applied | `Spanish` |
| `'Nope'`, `'explanatory'` | `system/init` reports them; nothing reaches the model; no error | The host check ("ignored") |
| String `systemPrompt` + style | The reminder is still there | Step 3 of the doc |
| `.claude/settings.json` vs `options.settings` | Both work; `options.settings` wins | Button 0, row 6 |
| Plugin style with `force-for-plugin` (+ `outputStyle: 'Explanatory'`) | `acme:House` applied; `system/init` says `default` / `Explanatory` | Button 0, row 7 ("replaced") |
| A subagent | Its request has no style reminder | Step 9 of the doc |
| One session: `applyFlagSettings`, a new file, `reloadOutputStyles`, `updateSettings`, `null` | Next turn: new init + new reminder; the old ones stay; `initializationResult()` stays stale; a new file is ignored until reload; `null` goes back to `options.settings` | Part C |

## Step 5: Design the concept

- **Real style files.** `styles-project/` (four project styles + `discount.js`), `styles-user/` (one user style),
  `styles-plugin/` (the `acme` plugin). Each run copies `styles-project` into `style-lab/runs/<id>`; the user style is
  copied into the fake `CLAUDE_CONFIG_DIR` (`style-lab/config`).
- **A wire tap** on its own port, one route prefix per run. It keeps each main-loop request's system prompt (without the
  billing header and the SDK identity line, with the run id hidden) and its `# Output Style` reminders.
- **A host check** (`verdict()`): the name asked for, the name `system/init` reports, and the name on the wire →
  default / applied / ignored / replaced.
- **Routes.** `GET /styles` (the files and their frontmatter), `POST /who` (7 setups in parallel), `POST /compare` (1–5
  styles in parallel, optional preset, optional custom style file), `POST /live` (one session, 5 turns), `GET /code`.
- **The tab.** The style files; button 0's table; one column per style (verdict, what the API got, the system-prompt diff
  against `default`, the answer); the turns of one session; a form for your own style; the code; a summary table.

## Step 6: Implement it

| File | What was done |
|---|---|
| `styles-project/`, `styles-user/`, `styles-plugin/` | New: the lab's style files and plugin |
| `server/concepts/34-output-styles.ts` | New: `newRun()`, `readStyle()`, the wire tap, `baseOptions()`, `systemSummary()`, `verdict()`, `collect()`, `writeCustom()`, `inbox()`, `live()`, the routes |
| `server/index.ts` | Mounted on `/api/c34` |
| `src/concepts/Concept34OutputStyles.tsx` | New: `StyleFiles`, `WhoTable`, `Columns`, `SystemBox`, `LiveTimeline`, Parts A to E, the custom style form |
| `src/App.tsx`, `src/styles.css` | Tab; `.where-*`, `.style-grid`, `.style-col`, `.v-*`, `.style-turn` |
| `.gitignore`, `Tab1-query().md` | `style-lab/`, table row |

`npx tsc --noEmit -p .` passed.

## Step 7: Test the routes

Port 3001 was left free for the other samples. A scratchpad script (`serve.ts`) mounted **only** the Concept 34 router
on port **3134**, with `.env` loaded and the `CLAUDE*` variables removed. A second script (`drive.mjs`) posted each body
the way the tab does and printed each SSE event on one line.

| Test | Result |
|---|---|
| `/compare`: `{}`, `styles: []`, a style twice, `"x;rm"`, an unknown key, a custom style not in `styles`, a custom named `concise`, `where: "etc"` | Refused with a clear message, nothing started |
| `/styles`, `/code` | 6 files with the right flags; 8 regions |
| `/who` | default · applied · ignored (case, with a hint) · ignored (no source) · applied · applied (settings.json) · replaced (acme:House) |
| Scenario 1 | 4 columns, all applied; the same 62-character system prompt |
| Scenario 2 | Pirate, Code reviewer (Verdict / Issues / Patch), Spanish: applied |
| Scenario 3 | Tutor: `# Doing tasks` removed (−3,264); Tutor (keeps coding): +57 |
| `/live` | 5 turns; turn 3 without a new reminder; `initializationResult()` stale |
| Custom style (project and user) | Written, applied; the user file was deleted after the run |

What the tests changed:

- **The system-prompt diff was noisy.** The preset contains the memory folder's path, which includes the run id, so
  every column showed that line as changed. The tap now replaces the run id with `<run>`. The diff's "first line" was the
  SDK's identity line, so the identity block is now left out when a preset follows it.
- **`updateSettings('localSettings')` was refused**: "the localSettings source is disabled for this session
  (--setting-sources)". The live session now loads `['user', 'project', 'local']`, and the doc and Part E say so.
- **The cost of each turn was the running total** (`total_cost_usd` adds up in one session). Each turn now shows its own part.
- **A request died silently** right after the live session: deleting the previous run's folder failed with `EPERM`
  (Windows: the closed Claude Code process still held it), and the error was thrown inside `newRun()`. That delete is
  now best effort, and the folder is removed by a later run.

## Step 8: Run it in the real app

Vite ran on port **5199** with a scratchpad config whose proxy pointed to the 3134 server, and headless Edge was driven
through the DevTools protocol (`shot.mjs`, its profile in the scratchpad), with real runs:

| Check | Page |
|---|---|
| Open tab 34 | The 6 style files and discount.js; clicking a name shows the file; 8 code buttons |
| Button 0 | 7 rows; the verdicts default, applied, ignored, ignored, applied, replaced, applied |
| Scenario 3 | 3 columns; "removed # Doing tasks" in red on Tutor; the system-prompt details |
| Scenario 4 | 5 turns; turn 3 in orange ("the style was ignored"); the host's control calls between turns |
| Console | No error |

The first browser run showed no turns for scenario 4, but that was the test script: it clicked the button while the
previous scenario was still running. The script now waits for the buttons to be enabled again. No code in the lab changed.

All test processes were stopped, and `style-lab/` and `.probe/` were deleted.

Costs: about $0.35 for all the probes, route tests and the browser test.

## Files added or changed

| File | Change |
|---|---|
| `styles-project/.claude/output-styles/*.md`, `styles-project/discount.js` | New: the project styles and the file the prompts ask about |
| `styles-user/output-styles/spanish.md` | New: the user style |
| `styles-plugin/` | New: the `acme` plugin with a forced style |
| `server/concepts/34-output-styles.ts` | New: the Concept 34 routes, the wire tap, the system-prompt diff, the host check |
| `server/index.ts` | Mounts `/api/c34` |
| `src/concepts/Concept34OutputStyles.tsx` | New: the Output styles tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Column, verdict and turn styles |
| `.gitignore` | `style-lab/` |
| `Tab1-query().md` | Adds Concept 34 to the table |
| `Tab34-Output-styles.md` | Explanation of the concept |
| `readme.md` | Same content as `Tab34-Output-styles.md` |
| `Build-steps.md` | This file |
