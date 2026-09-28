# MCP elicitation

This file explains Concept 33 (**MCP elicitation**) of the Claude Agent SDK Lab. Until now, an MCP tool got everything
it needed from the model's arguments (Concept 13). With **elicitation**, an MCP tool can stop halfway and **ask the
user**: the server sends an `elicitation/create` request back to its client, Claude Code. In the terminal, Claude Code
shows a form. With the SDK, **nothing is shown**: the request reaches **your** `onElicitation`, and your code must show
the form, wait for the person, and send back the answer.

**Goal:** build that form in your own UI from the server's JSON schema. Know the three answers you can send, who answers
when you don't (hooks, or nobody), what the server checks, and what this version of Claude Code does not support yet
(URL mode, in-process servers, answers from callback hooks).

| Concept | Topic | Routes |
|---|---|---|
| 33 | MCP elicitation: an MCP server asks the user mid-tool-call (`server.elicitInput()`), `Options.onElicitation` (`serverName`, `message`, `mode`, `requestedSchema`), `accept` + `content` / `decline` / `cancel`, a form built from the schema, the host's and the server's schema checks (`-32602`), a confirmation the server asks for itself, the client capabilities (`elicitation.form`, no `url` in 2.1.281) and a fallback for URL mode, the sign-in page, `system/elicitation_complete`, timeouts (`-32001`, the aborted `signal`), `Elicitation` / `ElicitationResult` hooks (command hooks answer, callback hooks only watch), in-process servers cannot elicit | `/api/c33/who` (SSE), `/run` (SSE), `/respond`, `/consent/:id`, `/log`, `/hooklog`, `/code` |

**Files touched:**

| File | Change |
|---|---|
| `mcp-servers/rooms-server.ts` | **New**: the lab's `rooms` MCP server (stdio). `book_room` and `cancel_booking` ask the user with a form, `connect_calendar` asks for a sign-in (URL mode, or a form fallback) |
| `elicit-hooks/hook.mjs` | **New**: a command hook. `autofill` answers the form itself, `policy` rewrites the user's answer |
| `server/concepts/33-mcp-elicitation.ts` | **New**: the options, `ask()` (the pending form), `/respond` and its schema check, the hooks, the sign-in page, the host check, the routes |
| `server/index.ts` | Mounts the router on `/api/c33` |
| `src/concepts/Concept33McpElicitation.tsx` | **New**: the tab (Parts A to D), the form card built from `requestedSchema` |
| `src/App.tsx` | Adds the tab |
| `src/styles.css` | The form card, server and hook rows |
| `.gitignore` | Ignores `elicit-lab/` |
| `Tab1-query().md` | Adds Concept 33 to the table |

---

## Step 1: A tool that asks the user

The `rooms` server is an ordinary stdio MCP server, like `notes-server.ts` in Concept 13. Claude Code starts it as a
child process. Nothing in the config "turns on" elicitation:

```ts
mcpServers: {
  rooms: { type: "stdio", command: process.execPath, args: ["--import", "tsx", "mcp-servers/rooms-server.ts"], env: { … } },
},
allowedTools: ["mcp__rooms__*"],
```

The model calls `book_room` with **no details**. The tool itself asks for them:

```ts
const r = await server.server.elicitInput({
  mode: "form",
  message: "Book a meeting room. Pick the room, the date and how many people will attend.",
  requestedSchema: {
    type: "object",
    properties: {
      room:      { type: "string", title: "Room", enum: ["Madrid", "Lisboa", "Roma"] },
      date:      { type: "string", title: "Date", format: "date" },
      attendees: { type: "integer", title: "Attendees", minimum: 1, maximum: 12 },
      projector: { type: "boolean", title: "Projector", default: false },
    },
    required: ["room", "date", "attendees"],
  },
});
if (r.action !== "accept") return text("The user declined the booking form. Nothing was booked.");
book(r.content);
```

The schema is **flat**: one object whose properties are strings (with `enum`, `format`: `date`, `email`, `uri`,
`date-time`, `minLength`, `maxLength`), numbers or integers (`minimum`, `maximum`) and booleans. No nested objects or
arrays. That is enough to draw a form without knowing the server.

The model **never sees the form or the values**. It only sees what the tool puts in its result
(`{"booked": {...}}`). So a server can ask for details that the model must not invent, or must not know.

## Step 2: The host answers in `onElicitation`

```ts
onElicitation: async (request, { signal }) => {
  // request: { serverName: "rooms", message, mode: "form", requestedSchema }
  return { action: "accept", content: { room: "Roma", date: "2026-10-20", attendees: 4, projector: true } };
}
```

| Return | Means | What the tool gets |
|---|---|---|
| `{ action: "accept", content }` | The user filled in the form | The content. The tool books the room |
| `{ action: "decline" }` | The user said **no** | No content. The tool books nothing and tells the model |
| `{ action: "cancel" }` | The user **dismissed** the form | The same, but a server may treat it as "ask again later" |

As in Concepts 31 and 32, the lab's `ask()` shows the request in the browser (a `form` event), keeps a promise in a map,
and returns it. `POST /respond` finds the promise and resolves it. The tab draws the form from `requestedSchema`: an
`enum` becomes a select, `format: "date"` a date picker, an integer a number field, a boolean a checkbox, and
`default` fills the first value.

**With no `onElicitation`, every elicitation is declined at once.** The tool gets `{ action: "decline" }` and the
user is never asked (button 0, row "no onElicitation").

## Step 3: Two checks of the answer

The host checks `content` against `requestedSchema` before it answers: a required field left empty, a value outside
the `enum`, a date that is not `YYYY-MM-DD`, a number above `maximum`. `POST /respond` answers **422** with the list,
and the form stays open so the user can fix it.

The server checks again. The MCP SDK's `elicitInput()` validates an `accept` against the schema it sent, and throws:

```
MCP error -32602: Elicitation response content does not match requested schema:
data/room must be equal to one of the allowed values, data/attendees must be <= 12
```

That error happens **inside the tool**. The lab's tool catches it and returns it as `is_error`, and the model then asks
the user in plain text for the details. To see it, press **Edit as JSON**, type `"attendees": 50`, and tick **skip the
host's check**.

Check in the host for a good form, and in the server for safety: the server never trusts its client.

## Step 4: The server asks for a confirmation

Scenario 2 starts with one booking (`a1b2c3`). The model calls `cancel_booking`, and the tool asks:

```ts
requestedSchema: {
  type: "object",
  properties: {
    confirm: { type: "boolean", title: "Cancel Roma on 2026-10-15?" },
    reason:  { type: "string", title: "Reason", minLength: 3, maxLength: 100 },
  },
  required: ["confirm", "reason"],
}
```

`allowedTools: ["mcp__rooms__*"]` lets the tool run with no permission prompt (Concept 4), yet the user is still
asked. This confirmation belongs to the **server**: it works with any host and any permission settings. The server
decides from the answer, not from the model: a `decline`, or an `accept` with the box unticked, keeps the booking.

## Step 5: What Claude Code supports (URL mode)

When the client connects, it says in `initialize` what it can do. The `client_capabilities` tool of the rooms server
shows it:

```json
{ "client": { "name": "claude-code", "version": "2.1.281" }, "elicitation": { "form": {} } }
```

MCP also has a **URL mode**: `elicitInput({ mode: "url", url, elicitationId })` asks the client to open a web page (a
sign-in, a payment), so the secret is typed on the **server's** page, never in the client. When the flow is over, the
server sends `notifications/elicitation/complete`, and Claude Code emits a `system/elicitation_complete` message. The
SDK types have all of it (`ElicitationRequest.url`, `elicitationId`, `SDKElicitationCompleteMessage`).

But Claude Code 2.1.281 declares **only `form`**. URL mode is behind a CLI feature flag that is off. So a URL
elicitation fails in the server before it is sent: `Client does not support url elicitation.`

`connect_calendar` shows the right way to handle this: **check the capabilities first**, and fall back:

```ts
if (server.server.getClientCapabilities()?.elicitation?.url) {
  await server.server.elicitInput({ mode: "url", message, url, elicitationId });   // then wait, then send "complete"
} else {
  // A form-only client: a form that carries the link (a link is not a secret) and a "done" box.
  await server.server.elicitInput({ mode: "form", message: `Open ${url}, sign in, then tick the box.`, requestedSchema: { … done: boolean … } });
}
const s = await fetch(`${url}/status`).then((r) => r.json());   // never trust the tick: ask YOUR page
```

The sign-in page is served by the lab (`/api/c33/consent/<id>`), and stands for the MCP server's own web page. The
password is typed there. Claude Code, the host and the model never see it. The tab shows the link in the form, and a
`sign-in page` row when you press Allow or Deny.

## Step 6: Who answers: hooks

Two hook events run around `onElicitation`. Button 0 runs the same `book_room` call in five setups, in parallel:

| Setup | `onElicitation` called | The server got |
|---|---|---|
| no `onElicitation` | no | `decline` |
| `onElicitation` | yes | `accept` (booked) |
| SDK **callback** hook on `Elicitation` that returns `accept` | no (none set) | `decline`: the hook ran, its answer was **ignored** |
| **command** hook on `Elicitation` (`settings.hooks`) + an `onElicitation` that declines | **no** | `accept` (booked by the hook) |
| an in-process `createSdkMcpServer` whose tool calls `elicitInput()` | no | error: `Client does not support form elicitation.` |

What the probes showed about Claude Code 2.1.281:

- **Order:** a command `Elicitation` hook → `onElicitation` → nothing (decline). Then a command `ElicitationResult` hook
  can **rewrite** the answer before the server gets it.
- **Only command hooks can answer.** A command hook is a program that Claude Code runs. It reads the hook input on stdin
  and prints its answer as JSON on stdout:

  ```json
  { "hookSpecificOutput": { "hookEventName": "Elicitation", "action": "accept", "content": { "room": "Lisboa", … } } }
  ```

  It goes in `settings.hooks` (the `settings` option, Concept 16), with `matcher` set to the MCP server's name.
  An SDK **callback** hook (`Options.hooks`, Concept 7) for the same events is called with the full input (server,
  message, schema, the user's answer), but what it returns is ignored. Use callbacks to **watch** and log, and command
  hooks to **decide**.
- **In-process servers cannot elicit.** For a `type: "sdk"` server, the client in the SDK declares no elicitation, so
  only external servers (stdio, http) can ask the user.

Scenario 4 uses both kinds. `hook.mjs policy` is a command hook on `ElicitationResult` with a company rule: more than 6
people must use Madrid. Answer **Roma, 8 attendees**. The host sent Roma, the hook changed it, the server got
**Madrid**, and the model reports Madrid. The callback hooks show two rows: they saw the form and your answer.

## Step 7: Time limits

`onElicitation` can wait for ever. The **server** does not: `elicitInput()` has a request timeout (60 s by default in
the MCP SDK; the rooms server sets 150 s with `ELICIT_TIMEOUT_MS`). When it fires, the tool gets
`MCP error -32001: Request timed out`, and Claude Code **aborts `onElicitation`'s `signal`**. The answer your host sends
after that is thrown away.

So the lab:

- gives up **before** the server, after 2 minutes, with `{ action: "cancel" }` (a countdown in the card)
- listens to `signal`, and closes the form with "the server gave up" if it fires
- answers `cancel` for every open form when the run ends or the browser disconnects

Never guess values for a user who did not answer.

## Step 8: The host check

After each run the lab compares three views, and trusts none of them alone:

- what the **host** answered (`onElicitation`'s results)
- what the **server** says it received (its log lines, posted to `/api/c33/log` from the other process)
- the bookings file, before and after

A difference between "host sent" and "server got" is shown in red: an `ElicitationResult` hook changed the answer on
the way (scenario 4), or the server refused it (`-32602`). As in Concepts 30 to 32: trust what your code saw, not what
the model says it did.

## Step 9: Part D, the summary

| You want | Do |
|---|---|
| A tool that asks the user | `server.server.elicitInput({ mode: "form", message, requestedSchema })` in an external MCP server |
| Show the form | `onElicitation`: draw fields from `request.requestedSchema`, show `request.message` and `request.serverName` |
| The user filled it in | `{ action: "accept", content }`, checked against the schema first |
| The user said no / closed it | `{ action: "decline" }` / `{ action: "cancel" }` |
| A confirmation the model cannot skip | Let the server ask for it (a boolean in the schema), whatever the host's permissions |
| A secret (password, OAuth) | URL mode, if the client declares `elicitation.url`; otherwise a form with the link, and the server checks its own page |
| Know what the client supports | `server.server.getClientCapabilities()?.elicitation` in the server |
| Answer without a person | A command `Elicitation` hook in `settings.hooks` |
| Change or veto the user's answer | A command `ElicitationResult` hook |
| Log every form and answer | SDK callback hooks on `Elicitation` / `ElicitationResult` (they watch; they cannot answer) |
| A time limit | Your own timer below the server's timeout, and watch `signal` |

## How to try it

1. `npm run dev`, then open the **33. MCP elicitation** tab. `ANTHROPIC_API_KEY` must be in `.env`.
2. Press **0 · Who answers the form?** and compare the five rows.
3. Press **1 · A form from the server**. Press **Accept** on the empty form (the host's errors), fill it in, and
   accept. Run it again and try **Decline**, **Cancel**, and **Edit as JSON** with `"attendees": 50` plus **skip the
   host's check**.
4. Press **2 · The server asks to confirm**. Accept with the box unticked: the booking stays.
5. Press **3 · Sign in on a web page**. Open the link in the form, press **Allow access** (or Deny) on that page, then
   tick the box and accept.
6. Press **4 · A hook rewrites your answer**. Answer Roma with 8 attendees, and read the host check.
7. Write your own job in the form (one booking, `a1b2c3`, exists).

Each scenario costs about $0.01 (Haiku 4.5), and button 0 about $0.03. A form waits for your answer for up to
2 minutes.
