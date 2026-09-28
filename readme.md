# Output styles

This file explains Concept 34 (**output styles**) of the Claude Agent SDK Lab. Concept 9 changed the agent with a
**system prompt**: you replace it, or append to Claude Code's own. An **output style** is a lighter tool. It changes
**how** the agent answers: its voice, the shape of each answer, whether it teaches. It does not change what the agent
can do (tools, permissions, the rest of the prompt). A style is a Markdown file that you **select by name**, so a
team can share a small set of styles, and a user can switch between them.

**Goal:** write a style file and select it from the SDK. Know where Claude Code looks for style files, what the model
really receives (a wire tap shows it), what `keep-coding-instructions` removes from the `claude_code` preset, how to
switch styles in a running session, and the silent failures a host must check for itself.

| Concept | Topic | Routes |
|---|---|---|
| 34 | Output styles: a style file (`name`, `description`, `keep-coding-instructions`, `force-for-plugin`), the `outputStyle` **setting** (there is no `outputStyle` option), the built-in styles (`default`, `Proactive`, `Concise`, `Explanatory`, `Learning`), project / user / plugin styles and `settingSources`, `system/init.output_style` and `available_output_styles`, what reaches the API (a `# Output Style` system-reminder, not the system prompt), what a style removes from the `claude_code` preset, silent failures (unknown name, wrong case, `force-for-plugin`), switching styles in one session (`applyFlagSettings`, `reloadOutputStyles`, `updateSettings('localSettings')`), a host-side check | `/api/c34/styles`, `/who` (SSE), `/compare` (SSE), `/live` (SSE), `/code` |

**Files touched:**

| File | Change |
|---|---|
| `styles-project/.claude/output-styles/*.md` | **New**: four project styles: `Pirate`, `Tutor`, `Tutor (keeps coding)`, `Code reviewer` |
| `styles-project/discount.js` | **New**: a small file with a bug, which the prompts ask about |
| `styles-user/output-styles/spanish.md` | **New**: a user style (`Spanish`), copied into the fake `CLAUDE_CONFIG_DIR` |
| `styles-plugin/` | **New**: the `acme` plugin with a forced style (`acme:House`) |
| `server/concepts/34-output-styles.ts` | **New**: the options, the wire tap, the system-prompt diff, the host check, `/who`, `/compare`, `/live` |
| `server/index.ts` | Mounts the router on `/api/c34` |
| `src/concepts/Concept34OutputStyles.tsx` | **New**: the tab (Parts A to E): the style files, the columns, one session's turns |
| `src/App.tsx` | Adds the tab |
| `src/styles.css` | Style badges, columns, verdict colors, turn rows |
| `.gitignore` | Ignores `style-lab/` |
| `Tab1-query().md` | Adds Concept 34 to the table |

---

## Step 1: A style is a Markdown file

```md
---
name: Code reviewer
description: Answers as a code review with a fixed shape. keep-coding-instructions true, because it still writes code.
keep-coding-instructions: true
---
Answer as a code review, in exactly this shape and nothing else:

Verdict: one line (OK, or the most serious problem).
Issues: a numbered list; each item is "file:line: the problem. Why it matters. The fix."
Patch: the corrected code in one code block, only the lines that change.
```

| Frontmatter | Meaning |
|---|---|
| `name` | The name you select. If missing, the file name without `.md`. **Case-sensitive** |
| `description` | Shown in lists (the `/output-style` menu of the terminal) |
| `keep-coding-instructions` | `true`: keep the coding rules of the `claude_code` preset (Step 5). Default `false` |
| `force-for-plugin` | Plugin styles only: the style is applied whenever the plugin is loaded (Step 6) |

Claude Code finds style files in three places, next to its five built-in styles:

| Where | Name | Found when |
|---|---|---|
| `<cwd>/.claude/output-styles/*.md` | `name` | `settingSources` includes `'project'` |
| `CLAUDE_CONFIG_DIR/output-styles/*.md` (normally `~/.claude/`) | `name` | `settingSources` includes `'user'` |
| `<plugin>/output-styles/*.md` | `<plugin>:<name>` | the plugin is in `plugins` |
| built-in | `default`, `Proactive`, `Concise`, `Explanatory`, `Learning` | always |

The lab's files are real files you can open: `styles-project/`, `styles-user/`, `styles-plugin/`. Each run works on its
own copy of `styles-project` (`style-lab/runs/<run id>`), and the fake `CLAUDE_CONFIG_DIR` is `style-lab/config`.

## Step 2: Select a style: it is a setting

`Options` has no `outputStyle` field. `outputStyle` is a **setting**, like `permissions` or `plansDirectory`
(Concept 16), so you set it the same way:

```ts
const options: Options = {
  cwd: run.work,                          // project styles: <cwd>/.claude/output-styles/
  settings: { outputStyle: "Code reviewer" }, // the "flag" layer: the highest one
  settingSources: ["user", "project"],    // without these, project and user styles are not found
  env: { ...env, CLAUDE_CONFIG_DIR },     // user styles: CLAUDE_CONFIG_DIR/output-styles/
};
```

You can also write `"outputStyle": "Code reviewer"` in `.claude/settings.json`, `.claude/settings.local.json` or the
user's `settings.json`, which is what the terminal's `/output-style` and `/config` do. `options.settings` wins over
all of them.

The session tells you what it selected:

```ts
if (m.type === "system" && m.subtype === "init") {
  m.output_style;                                              // "Code reviewer"
  (await q.initializationResult()).available_output_styles;   // ["default", "Proactive", …, "Code reviewer", …]
}
```

**Parts A1 and A2** run the same prompt ("Read discount.js. Is there a bug?") in four styles in parallel. The fix is the
same in every column (`price * percent / 100`). The answers differ. Explanatory is told to add `★ Insight` boxes and Learning to
leave a `TODO(human)` for you to write (open "what the API got" to read their texts; a one-question prompt does not always show them). Concise cuts the words, Code reviewer gives a Verdict / Issues / Patch,
Pirate says "Arr!", and Spanish answers in Spanish.

## Step 3: What the model gets

The lab sets `ANTHROPIC_BASE_URL` to a small **wire tap** (as in Concept 29). It forwards every request to the API and
keeps a copy of each main-loop request. This is what a style sends. The style text is **not** in the system prompt.
It comes in the **first user message**, as two system-reminders:

```
<system-reminder>
# Output Style: Code reviewer
Answer as a code review, in exactly this shape and nothing else: …
</system-reminder>
<system-reminder>
Code reviewer output style is active. Remember to follow the specific guidelines for this style.
</system-reminder>
```

Because of that:

- A style works with the SDK's short default system prompt, with the `claude_code` preset, and with your own string
  `systemPrompt`. (Without a preset, all four columns of A1 had the same 62-character system prompt.)
- The style is a message in the history. This matters when you switch styles later (Step 7).

## Step 4: The silent failures (button 0)

**0 · Which style really applied?** runs one short prompt in 7 setups. For each one it shows what `system/init` says
and what the API really got:

| Setup | `system/init.output_style` | The API got | Host check |
|---|---|---|---|
| no `outputStyle` | `default` | no style | default |
| `'Explanatory'` | `Explanatory` | `Explanatory` | applied |
| `'explanatory'` (lowercase) | `explanatory` | **nothing** | **ignored**: names are case-sensitive |
| `'Pirate'`, `settingSources: []` | `Pirate` | **nothing** | **ignored**: the project folder was not read |
| `'Pirate'`, `settingSources: ['user', 'project']` | `Pirate` | `Pirate` | applied |
| `.claude/settings.json` `{ outputStyle: 'Code reviewer' }` | `Code reviewer` | `Code reviewer` | applied |
| `plugins: [acme]` + `'Explanatory'` | `Explanatory` | **`acme:House`** | **replaced** by the forced plugin style |

An unknown name gives **no error and no warning**, and `system/init` still reports the name you asked for. So the
host checks it itself:

```ts
function verdict(asked, init, available, calls) {
  const got = calls.at(-1)?.reminders.at(-1)?.name;          // the latest "# Output Style:" on the wire
  if (!got) return init === "default" ? "default" : "ignored"; // not in available_output_styles
  return got === init ? "applied" : "replaced";
}
```

Without a wire tap, check `available_output_styles.includes(name)` before you start the session.

## Step 5: What a style changes in the system prompt (`keep-coding-instructions`)

**Part B** (`3 · keep-coding-instructions`) uses `systemPrompt: { type: "preset", preset: "claude_code" }` and
compares the system prompt with the `default` column, line by line. `Tutor` and `Tutor (keeps coding)` have the **same
text**. Only the frontmatter differs:

| Column | System prompt | What changed |
|---|---|---|
| `default` | 26,584 chars | — |
| `Tutor` | 23,320 chars (−3,264) | The first line, and **the whole `# Doing tasks` section is removed** (16 lines) |
| `Tutor (keeps coding)` | 26,641 chars (+57) | Only the first line |

The first line changes with **any** style:

```
- You are an interactive agent that helps users with software engineering tasks. …
+ You are an interactive agent that helps users according to your "Output Style", which describes how you should respond to user queries. …
```

`# Doing tasks` holds Claude Code's coding rules: don't add features or abstractions beyond the task, no comments by
default, no needless error handling, watch for security holes, prefer editing files to creating new ones. A style for
a **coding** agent should say `keep-coding-instructions: true`. A style that turns the agent into something else (a
tutor, a writer) can leave it out. The built-in `Explanatory` and `Learning` keep them.

## Step 6: A plugin can force its style

```md
---
name: House
force-for-plugin: true
---
Write in the ACME house style: … End every answer with the line "— ACME Engineering".
```

A plugin's styles are named `<plugin>:<name>` (`acme:House`), and you can select them like any other style. With
`force-for-plugin: true`, the style is applied **whenever the plugin is loaded**, whatever `outputStyle` says. In
row 7 of button 0, `system/init` said `Explanatory`, but the API got `acme:House`, and the answer ended with
"— ACME Engineering". Outside a plugin, `force-for-plugin` is ignored (the CLI logs a warning).

## Step 7: Switching styles in one session

**Part C** (`4 · Switch mid-session`) runs one session with streaming input (Concept 12), five turns. Between turns
the host calls the `Query`'s control methods (Concept 26):

| Before the turn | The turn's `system/init` | New style reminder on the wire | Answer |
|---|---|---|---|
| (start) `settings: { outputStyle: 'Pirate' }` | `Pirate` | `Pirate` | "Arr! I be Claude…" |
| `q.applyFlagSettings({ outputStyle: 'Concise' })` | `Concise` | `Concise` | One plain sentence |
| writes `.claude/output-styles/haiku.md`, `applyFlagSettings({ outputStyle: 'Haiku' })` | `Haiku` | **none**: ignored | Still plain |
| `q.reloadOutputStyles()`, then the same `applyFlagSettings` | `Haiku` | `Haiku` | A haiku |
| `q.updateSettings('localSettings', { outputStyle: 'Explanatory' })`, `applyFlagSettings({ outputStyle: null })` | `Pirate` | `Pirate` | "Arr! Fair winds…" |

What this shows:

- **The switch takes effect on the next turn.** Each turn starts with a new `system/init`, and the new style arrives as
  a **new** reminder in that turn's user message.
- **The old reminders stay in the history** (`Pirate → Concise → Haiku → Pirate`). The model sees all of them, and its
  own earlier answers too, so styles can mix. For a clean switch, start a new session (a fork or a resume keeps the old history).
- **`initializationResult()` is cached.** After `applyFlagSettings`, `(await q.initializationResult()).output_style`
  still said `Pirate`. Read `system/init` instead.
- **A style file written while the session runs is invisible** until `q.reloadOutputStyles()`. Before that, selecting it
  is silently ignored, as in Step 4. `reloadOutputStyles()` returns the new list.
- **`updateSettings('localSettings', …)`** saves the choice the way `/config` does, to `<cwd>/.claude/settings.local.json`.
  It is **refused** ("the localSettings source is disabled for this session") unless `settingSources` includes
  `'local'`, so this session uses `['user', 'project', 'local']`.
- **The layers:** `applyFlagSettings({ outputStyle: null })` removes the value it set. The session then went back to
  `options.settings` (`Pirate`), not to the `Explanatory` just saved in `settings.local.json`: `options.settings` is
  still the higher layer. The saved value is for the **next** session that does not set `options.settings`.

## Step 8: Your own style

The form under Part C writes your style to a real file before the run: to `<run>/.claude/output-styles/` (project) or to
`CLAUDE_CONFIG_DIR/output-styles/` (user; the file name carries the run id and is deleted after the run). Then it runs
the prompt next to `default`. Tick **keep-coding-instructions** and **the `claude_code` preset** to see Step 5 with your
own file. "Show the file that will be written" shows the exact file first.

The server checks the body with zod: a name of letters, digits, spaces and `. ( ) - _`, not a built-in name, and at
most 3000 characters of instructions.

## Step 9: Subagents

A subagent (the `Agent` tool, Concept 8) does **not** get the style. In a probe with `Pirate`, the main agent's
requests carried the `# Output Style: Pirate` reminder, and the subagent's request did not. The main agent still
repeated the subagent's answer in its pirate voice. A subagent's voice comes from its own `prompt`.

## Step 10: Part E, the summary

| You want | Do |
|---|---|
| Change the voice or shape of the answers | An output style, selected with `settings: { outputStyle: 'Name' }` |
| Replace what the agent is | A `systemPrompt` (Concept 9), or an agent (`agent` option) |
| Share styles in a repo | `.claude/output-styles/*.md` + `settingSources` with `'project'` |
| One style for all of a user's projects | `CLAUDE_CONFIG_DIR/output-styles/*.md` + `'user'` |
| A company style everyone gets | A plugin with `force-for-plugin: true` |
| A coding agent with a style | `keep-coding-instructions: true` |
| Know which style is selected | `system/init.output_style` |
| Know it was applied | `available_output_styles.includes(name)`, or look at the request (`# Output Style:`) |
| Switch during a session | `q.applyFlagSettings({ outputStyle })`, from the next turn on (the old reminders stay) |
| A style file added mid-session | `q.reloadOutputStyles()` first |
| Save the user's choice | `q.updateSettings('localSettings', { outputStyle })` with `'local'` in `settingSources` |

## How to try it

1. `npm run dev`, then open the **34. Output styles** tab. `ANTHROPIC_API_KEY` must be in `.env`.
2. Open the style files at the top (click a name) to see the frontmatter.
3. Press **0 · Which style really applied?** and read the host check column: two rows are ignored, one is replaced.
4. Press **1 · The built-in styles** and **2 · Your own style files**. In each column open **what the API got**.
5. Press **3 · keep-coding-instructions** and open **system prompt** in each column.
6. Press **4 · Switch mid-session** and follow the turns: turn 3 is ignored (orange), and the style history grows.
7. Write your own style in the form, and run it with and without `keep-coding-instructions` and the preset.

Each button costs about $0.02 (Haiku 4.5), and button 3 about $0.04, because it sends the full preset.
