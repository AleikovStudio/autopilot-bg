# Mixed Execution Mode (agent/human) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let Autopilot tickets carry an `owner: agent | human` field so a plan can mix subagent-executed work with work the user does by hand (e.g. clicking in a WordPress/Divi 5 admin), tracked on the same dashboard and manifest — plus a manual-mode requirement-confirmation pass in Briefing.

**Architecture:** Pure additive change to the Autopilot skill's own instruction files (`phases/*.md`) and its dashboard template (`phases/dashboard-template.html`). No application code, no build step, no package manager — every "file" here is either a markdown rulebook read by an LLM at the start of a phase, or one self-contained HTML+JS file that renders `state.json`. Backward compatibility is enforced structurally: every read of the new `owner` field falls back to `"agent"`, so any `state.json` written before this change (or by anyone who hasn't updated) renders identically to today.

**Tech Stack:** Markdown (instruction files), vanilla JS embedded in a single HTML file (dashboard, no framework, no bundler), Node.js only as a syntax-checking tool during this implementation (never a runtime dependency of the shipped skill).

## Global Constraints

- Every new field defaults safely when absent — `owner` missing on a ticket reads as `"agent"` everywhere (design doc, section "Модел на данните"). Never write code that treats a missing `owner` as an error.
- No servers, ever. The dashboard stays a single static file opened by double-click or `file://`. Do not add a build step, a dev server, or a write-back mechanism (design doc, "Извън обхвата").
- All user-facing strings are Bulgarian-first with an English counterpart in the same `I18N` object shape already used in `dashboard-template.html` (see existing `bg:{...}` / `en:{...}` keys).
- Every phase-file edit must read correctly in isolation — phase files are read "at the moment that phase starts, not before" (SKILL.md), so a rule split across two files must not assume the reader has the other file's context loaded.
- Source of truth for exact wording: `docs/superpowers/specs/2026-08-10-mixed-execution-mode-design.md`. Where this plan quotes it, the quote is authoritative; do not paraphrase further when implementing.

---

### Task 1: `owner` field — data model + dashboard rendering

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\7-instruments.md` (state.json ticket example, ~line 119-137)
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\dashboard-template.html` (I18N object ~line 277-351, `statusColor`/rows builder ~line 520-580, ticket table rows ~line 571-580, "Сега тече" block ~line 529-549)

**Interfaces:**
- Consumes: nothing from earlier tasks (this is the foundation).
- Produces: `t.owner` field convention (`"agent" | "human" | undefined`, treat `undefined` as `"agent"`) that Tasks 2–4 rely on when writing tickets; `L.ownerAgent` / `L.ownerHuman` / `L.waitingForYou` i18n keys that Task 3's chat-facing text can reference by name when describing what the dashboard shows.

- [ ] **Step 1: Add `owner` to the documented ticket shape in `7-instruments.md`**

Open `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\7-instruments.md`. Find the ticket example inside the `state.json` code block:

```json
  "tickets": [
    {
      "id": "03",
      "title": "Приемане на заявка от клиент",
      "requirements": ["R01", "R01.1", "A01"],
      "blockedBy": ["01", "02"],
      "wave": 2,
      "zone": ["src/bot/"],
      "status": "done",
```

Add `"owner": "agent",` immediately after `"id": "03",` so the example reads:

```json
  "tickets": [
    {
      "id": "03",
      "owner": "agent",
      "title": "Приемане на заявка от клиент",
      "requirements": ["R01", "R01.1", "A01"],
      "blockedBy": ["01", "02"],
      "wave": 2,
      "zone": ["src/bot/"],
      "status": "done",
```

Immediately below the existing line `` `memoryFile` is the project memory chosen in Phase 0 — `CLAUDE.md` or `AGENTS.md`, see `phases/9-memory.md`. A resume reads that file first. `` add a new paragraph:

```markdown
**`owner` says who executes the ticket** — `"agent"` (subagent writes code/acts through a connected tool, unchanged from today) or `"human"` (the user does it by hand — WordPress admin clicks, Divi Builder, anything outside the agent's reach). **Missing `owner` reads as `"agent"` everywhere it is checked** — this is what keeps every run written before this field existed rendering exactly as it did before. See `phases/4-plan.md` for how it gets decided and `phases/5-subagents.md` for how Phase 5 branches on it.
```

- [ ] **Step 2: Verify the doc edit**

Run: `grep -n "owner" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\7-instruments.md"`
Expected: two hits — the JSON field and the new prose paragraph.

- [ ] **Step 3: Add i18n labels to `dashboard-template.html`**

Open `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\dashboard-template.html`. In the `bg:{...}` object, find:

```
    reqs:"Изисквания",reqsSub:(d,f,p)=>`отпаднали ${d} · отложени ${f} · заместители ${p}`,
    reqsTitle:"Изисквания от брифа",thReqId:"№",thReqText:"Изискване",thReqStatus:"Статус",
    reqStatus:{open:"отворено",'in-spec':"в спецификацията",'in-ticket':"в подзадача",done:"готово",
      placeholder:"заместител",deferred:"отложено",dropped:"отпаднало"},
    progress:"Ход на разработката",
```

Replace with (adds three new keys, everything else unchanged):

```
    reqs:"Изисквания",reqsSub:(d,f,p)=>`отпаднали ${d} · отложени ${f} · заместители ${p}`,
    reqsTitle:"Изисквания от брифа",thReqId:"№",thReqText:"Изискване",thReqStatus:"Статус",
    reqStatus:{open:"отворено",'in-spec':"в спецификацията",'in-ticket':"в подзадача",done:"готово",
      placeholder:"заместител",deferred:"отложено",dropped:"отпаднало"},
    ownerAgent:"агент",ownerHuman:"ти",waitingForYou:"Чака теб",
    progress:"Ход на разработката",
```

In the `en:{...}` object, find the mirror block:

```
    reqs:"Requirements",reqsSub:(d,f,p)=>`dropped ${d} · deferred ${f} · placeholders ${p}`,
    reqsTitle:"Requirements from the brief",thReqId:"#",thReqText:"Requirement",thReqStatus:"Status",
    reqStatus:{open:"open",'in-spec':"in spec",'in-ticket':"in ticket",done:"done",
      placeholder:"placeholder",deferred:"deferred",dropped:"dropped"},
    progress:"Build progress",
```

Replace with:

```
    reqs:"Requirements",reqsSub:(d,f,p)=>`dropped ${d} · deferred ${f} · placeholders ${p}`,
    reqsTitle:"Requirements from the brief",thReqId:"#",thReqText:"Requirement",thReqStatus:"Status",
    reqStatus:{open:"open",'in-spec':"in spec",'in-ticket':"in ticket",done:"done",
      placeholder:"placeholder",deferred:"deferred",dropped:"dropped"},
    ownerAgent:"agent",ownerHuman:"you",waitingForYou:"Waiting on you",
    progress:"Build progress",
```

- [ ] **Step 4: Add an owner badge helper next to `statusColor`**

Find:

```js
  const statusColor = s => s === 'done' ? 'var(--ok)' : s === 'failed' ? 'var(--bad)'
    : s === 'in-progress' ? 'var(--warn)' : 'var(--line)';

  const reqStatusColor = s => s === 'done' ? 'var(--ok)' : s === 'dropped' ? 'var(--bad)'
    : (s === 'in-spec' || s === 'in-ticket' || s === 'placeholder') ? 'var(--warn)' : 'var(--line)';
```

Add immediately after:

```js
  const ownerBadge = t => {
    const isHuman = t.owner === 'human';
    return `<span class="tag" title="${isHuman ? L.ownerHuman : L.ownerAgent}">${isHuman ? '👤' : '🤖'}</span>`;
  };
```

`t.owner === 'human'` is the only place ownership is branched on for rendering — everything else (`undefined`, `'agent'`, any future typo) falls through to the agent icon, matching the "missing owner reads as agent" rule.

- [ ] **Step 5: Show the badge in the tickets table**

Find the `rows` builder:

```js
  const rows = T.map(t => `<tr>
    <td><b>${esc(t.id)}</b></td>
    <td><span class="dot" style="background:${statusColor(t.status)}"></span>${esc(t.title)}
      ${t.concerns?.length ? `<div class="sub">${esc(t.concerns.join('; '))}</div>` : ''}</td>
```

Replace the second `<td>` line with:

```js
    <td>${ownerBadge(t)}<span class="dot" style="background:${statusColor(t.status)}"></span>${esc(t.title)}
      ${t.concerns?.length ? `<div class="sub">${esc(t.concerns.join('; '))}</div>` : ''}</td>
```

- [ ] **Step 6: Show "Чака теб" instead of a subagent clock for waiting human tickets**

Find, inside the `timeline` builder:

```js
    const rows = inWave.map(t => {
      const active = t.status === 'in-progress', pend = t.status === 'pending';
      // при незапочната подзадача часовник няма: „0:00“ се чете като „върви и не мърда“
      const clock = (t.startedAt && !pend) ? timerHTML(t.startedAt, t.finishedAt) : '—';
```

Replace with:

```js
    const rows = inWave.map(t => {
      const active = t.status === 'in-progress', pend = t.status === 'pending';
      const waitingOnHuman = active && t.owner === 'human';
      // при незапочната подзадача часовник няма: „0:00“ се чете като „върви и не мърда“
      const clock = (t.startedAt && !pend) ? timerHTML(t.startedAt, t.finishedAt) : '—';
```

Then find the line that renders each row's title bar (a few lines below, ends with `${esc(t.title)}</div></div>`):

```js
          style="width:100%${pend?'':';background:'+statusColor(t.status)}">${esc(t.title)}</div></div>
        <div class="mins">${clock}</div></div>`;
```

Replace with:

```js
          style="width:100%${pend?'':';background:'+statusColor(t.status)}">${ownerBadge(t)}${esc(t.title)}</div></div>
        <div class="mins">${waitingOnHuman ? `<span class="badge">${L.waitingForYou}</span>` : clock}</div></div>`;
```

- [ ] **Step 7: Syntax-check the extracted script**

The file is HTML with one `<script>...</script>` block holding all the JS. Extract and check it:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:/Users/user/PhpstormProjects/autopilot-bg/skills/autopilot/phases/dashboard-template.html', 'utf8');
const m = html.match(/<script>([\s\S]*)<\/script>/);
fs.writeFileSync('/tmp/dashboard-check.js', m[1]);
"
node --check /tmp/dashboard-check.js
```

Expected: no output from `node --check` (silent success means valid syntax). If it prints a `SyntaxError`, the template literal or brace nesting from Steps 4-6 is broken — re-check the exact `old_string`/`new_string` match before retrying.

- [ ] **Step 8: Render a sample `owner` state and eyeball it**

```bash
mkdir -p /tmp/owner-render-check/.autopilot
cp "C:/Users/user/PhpstormProjects/autopilot-bg/skills/autopilot/phases/dashboard-template.html" /tmp/owner-render-check/.autopilot/dashboard.html
```

Edit the copied file's `const STATE = {...}` line (between `/*autopilot:state:start*/` and `/*autopilot:state:end*/`) to a minimal state with one `agent` and one `human` ticket, both `in-progress`, in the same wave:

```json
{"slug":"t","title":"Owner render check","mode":"manual","depth":"normal","tier":"T1","briefFile":"x.md","memoryFile":"CLAUDE.md","startedAt":"2026-08-10T10:00:00+03:00","updatedAt":"2026-08-10T10:00:00+03:00","finishedAt":null,"stages":[{"id":"preflight","status":"done","startedAt":"2026-08-10T10:00:00+03:00","finishedAt":"2026-08-10T10:00:00+03:00"},{"id":"manifest","status":"done"},{"id":"briefing","status":"done"},{"id":"spec","status":"done"},{"id":"plan","status":"done"},{"id":"build","status":"active","startedAt":"2026-08-10T10:00:00+03:00"},{"id":"review","status":"pending"},{"id":"final","status":"pending"}],"requirements":{"total":2,"done":0,"inTicket":2,"inSpec":0,"placeholder":0,"deferred":0,"dropped":0},"tickets":[{"id":"01","owner":"agent","title":"Agent ticket","requirements":["R01"],"blockedBy":[],"wave":1,"zone":["src/"],"status":"in-progress","startedAt":"2026-08-10T10:00:00+03:00","retries":0},{"id":"02","owner":"human","title":"Human ticket","requirements":["R02"],"blockedBy":[],"wave":1,"zone":["wp-admin"],"status":"in-progress","startedAt":"2026-08-10T10:00:00+03:00","retries":0}],"singlePass":null,"tests":null,"debt":{"placeholders":[],"assumptions":[],"emptyEnv":[]},"additions":[],"blind":null}
```

Open `/tmp/owner-render-check/.autopilot/dashboard.html` in a browser (`Start-Process` on Windows, `open` on macOS, `xdg-open` on Linux). Confirm visually:
- Ticket "Agent ticket" shows 🤖 next to its title.
- Ticket "Human ticket" shows 👤 next to its title, and its wave row says "Чака теб" (or "Waiting on you" if you toggle EN) instead of a running clock.
- No console errors (open devtools, check the Console tab).

Delete the scratch folder afterward: `rm -rf /tmp/owner-render-check`.

- [ ] **Step 9: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/7-instruments.md skills/autopilot/phases/dashboard-template.html
git commit -m "Добавено поле owner (agent/human) към тикетите + визуализация в таблото"
```

---

### Task 2: Phase 4 (Plan) — decide and show ownership

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\4-plan.md`

**Interfaces:**
- Consumes: `owner` field convention from Task 1 (no code dependency — this is a documentation-only task that tells the LLM orchestrator to *write* the field).
- Produces: the rule "every published ticket carries `owner`" that Task 3 (Phase 5) and Task 4 (Phase 0 resume) both assume is already true by the time their logic runs.

- [ ] **Step 1: Add the ownership decision table**

Open `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\4-plan.md`. Find:

```markdown
## Publishing the plan to the instruments
```

Insert a new section immediately **before** it:

```markdown
## Who executes each ticket

Every ticket gets an `owner`: `agent` (default — a subagent builds it, exactly as today) or `human` (the user does it by hand — a WordPress admin setting, a Divi Builder change, anything outside what the agent's tools reach).

| Signal | Owner |
|---|---|
| Buildable with a tool the agent has — code in the repo, a connected MCP (e.g. Respira) | `agent` |
| Requires a manual action only the user can perform — builder UI clicks, an admin toggle with no matching MCP ability | `human` |
| Ambiguous | default to `agent`, but flag it plainly when showing the plan — do not silently guess on an ambiguous one |

In **manual** mode, showing the plan (below) includes each ticket's owner, and the user can reassign any ticket before saying „ок“ — same gate, same moment, no extra round-trip.

In **full** and **semi**, ownership is decided from the table above without waiting for confirmation, in keeping with those modes only pausing on blocking unknowns — and it is reported like any other decision made for the user.

```

- [ ] **Step 2: Add `owner` to the published ticket JSON example**

Find:

```json
{ "id": "04", "title": "Панел на майстора: опашка от заявки", "requirements": ["R04", "R04.1"],
  "blockedBy": ["02"], "wave": 3, "zone": ["src/admin/"], "status": "pending", "retries": 0 }
```

Replace with:

```json
{ "id": "04", "owner": "agent", "title": "Панел на майстора: опашка от заявки", "requirements": ["R04", "R04.1"],
  "blockedBy": ["02"], "wave": 3, "zone": ["src/admin/"], "status": "pending", "retries": 0 }
```

- [ ] **Step 3: Add `owner` to the ticket file template**

Find the ticket file example (starts `# 03 — Приемане на заявка от клиент`):

```markdown
# 03 — Приемане на заявка от клиент

**Изисквания:** R01, R01.1, A01
**Blocked by:** 01, 02
**Зона:** `src/bot/`
**Вълна:** 2
**Status:** ready
```

Replace with:

```markdown
# 03 — Приемане на заявка от клиент

**Изисквания:** R01, R01.1, A01
**Blocked by:** 01, 02
**Зона:** `src/bot/`
**Вълна:** 2
**Изпълнява:** агент
**Status:** ready
```

(`**Изпълнява:** ти` for a `human`-owned ticket file.)

- [ ] **Step 4: Update the "Showing the plan" section's manual-mode line**

Find:

```markdown
**manual** — the plan is a gate. Show it with technical detail, discuss granularity and order, adjust on request, wait for an explicit „ок“. Phase 5 starts only on agreed tickets.
```

Replace with:

```markdown
**manual** — the plan is a gate. Show it with technical detail, **including who executes each ticket**, discuss granularity/order/ownership, adjust on request, wait for an explicit „ок“. Phase 5 starts only on agreed tickets.
```

- [ ] **Step 5: Verify**

Run: `grep -n "owner\|Изпълнява" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\4-plan.md"`
Expected: 4 hits — the decision table's header row mention, the JSON example, the ticket file template, and no stray leftover text from the old lines (diff the file against git if unsure: `git diff skills/autopilot/phases/4-plan.md`).

- [ ] **Step 6: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/4-plan.md
git commit -m "Фаза 4: тикетите вече получават owner (agent/human), показва се в ръчния гейт"
```

---

### Task 3: Phase 5 (Subagents) — branch on ticket ownership

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\5-subagents.md`

**Interfaces:**
- Consumes: `owner` field written by Phase 4 (Task 2).
- Produces: the "human ticket → no subagent, chat-confirmed done" behavior that Task 5 (Phase 6) needs to know to skip, and that Task 4 (Phase 0 resume) needs to know not to reset.

- [ ] **Step 1: Add a human-ticket branch to "Before each ticket"**

Find:

```markdown
## Before each ticket

Set the ticket's `status` to `in-progress` and its `startedAt` to now **before** launching the subagent, and mirror it into the dashboard. It costs one edit, and it is the difference between the user watching a ticket run and the user watching nothing happen for eighteen minutes.

For a wave, that is **one state write for the whole wave**, before the launch message — all of its tickets flipped together. Two clocks running side by side on the dashboard is what parallel work looks like; two tickets marked `in-progress` an edit apart is the same thing and costs half as much.
```

Replace with:

```markdown
## Before each ticket

Set the ticket's `status` to `in-progress` and its `startedAt` to now **before** launching the subagent, and mirror it into the dashboard. It costs one edit, and it is the difference between the user watching a ticket run and the user watching nothing happen for eighteen minutes.

For a wave, that is **one state write for the whole wave**, before the launch message — all of its tickets flipped together. Two clocks running side by side on the dashboard is what parallel work looks like; two tickets marked `in-progress` an edit apart is the same thing and costs half as much.

**A `human`-owned ticket does not get a subagent.** Mark it `in-progress` and `startedAt` exactly like an `agent` ticket — the clock still matters, the user should see how long they have been waiting on themselves — but do not launch anything for it. Instead, in the same message where you report the wave's launch, add one line naming what the user needs to do: „Готов си за 03 — настрой навигационното меню в Divi 5 Theme Builder по спецификацията. Кажи ми като приключиш.“ A wave with both kinds in it launches its `agent` tickets normally and announces its `human` tickets in the same breath — neither waits for the other.
```

- [ ] **Step 2: Add a completion path for human tickets to "After each ticket"**

Find the numbered "After each ticket" list (starts `1. **Read the contract block.**`). Insert a new subsection immediately **after** the numbered list and **before** `### When two tickets return together`:

```markdown
### When a ticket is `human`-owned

There is no contract block, no diff, no test run — the user did the work outside the repo. When they tell you it is done (by ticket id, by name, or unambiguously — "готово" when exactly one human ticket is waiting):

1. **Update the manifest** — the ticket's requirements move to `done`, Основание notes it was done by the user, no commit reference.
2. **Skip Phase 6 entirely.** There is nothing to review — see `phases/6-review.md`.
3. **Update the instruments** — `status: "done"`, `finishedAt` now, leave `files`/`tests`/`commit` as they were (empty). The dashboard already renders these as `—` when absent; that is correct here, not a gap to fill.
4. **Tell the user one plain-language line**, same as an agent ticket: „Менюто е готово — 3 от 8 подзадачи.“
5. If the user instead says it is **not** done, or only partly — do not mark it done. Ask what is missing and leave it `in-progress`. This is not a `failed` ticket (nothing failed); it is simply still open. Only mark `failed` if the user explicitly abandons the ticket, and treat that the same as any dropped requirement: back to the manifest, a quote, a question — never a silent status flip.

A `human` ticket never produces a `D##` row or an `interfaces.md` entry on its own — those exist for what code exposes to other code. If the user's manual work changes a decision the plan assumed (e.g. a menu structure that turned out not to fit the theme), that is still a plan-contradiction per "When the build contradicts the plan" below, handled the same way regardless of which kind of ticket surfaced it.
```

- [ ] **Step 3: Add the mid-run ownership handoff rule**

Find the section `## When a ticket fails` (the design doc's edge case about reassigning ownership belongs near it, since both describe an in-flight ticket changing course). Insert a new section immediately **before** `## When a ticket fails`:

```markdown
## When the user takes over a ticket already running

Allowed at any point, the same way a mode or depth switch is allowed mid-run: the user says something like „дай на мен тази задача“ for a ticket currently `in-progress` with `owner: "agent"`.

1. If a subagent is in flight for it, let it finish its current step rather than killing it mid-write — an interrupted code edit is worse than a finished one you then discard.
2. Set `owner` to `"human"` on that ticket.
3. Whatever the subagent already produced stays — this is a handoff, not a rollback. Tell the user in one line what exists so far and what is left: „Досега е готова схемата на менюто във файла; остава да я пренесеш в Theme Builder.“
4. From here it follows the `human`-ticket path exactly: no further subagent launches for it, marked `done` when the user says so, no Phase 6 review.

The reverse direction — a `human` ticket the user asks the agent to take over instead — follows the same shape: flip `owner` to `"agent"`, then launch a subagent for it exactly as Phase 5 would for any newly-launchable ticket, with the user's own description of what is already done included in its prompt so it does not redo finished work.
```

- [ ] **Step 4: Verify**

Run: `grep -n "human" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\5-subagents.md"`
Expected: multiple hits across the three new blocks added in Steps 1, 2 and 3, none elsewhere (confirms nothing was duplicated).

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/5-subagents.md
git commit -m "Фаза 5: human-тикети не пускат субагент, отмятат се по думата на потребителя"
```

---

### Task 4: Phase 0 (Preflight) — resume must not reset human tickets

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\0-preflight.md`

**Interfaces:**
- Consumes: `owner` field (Task 2), human-ticket completion rules (Task 3).
- Produces: nothing further downstream — this is a leaf fix.

- [ ] **Step 1: Split the resume rule by ticket owner**

Open `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\0-preflight.md`. Find, inside "Resuming an interrupted flight":

```markdown
3. A ticket marked `in-progress` in `state.json` with no commit behind it was interrupted mid-flight. Reset it to `pending` and run it again from scratch — a half-applied ticket is worse than a fresh one.
```

Replace with:

```markdown
3. An **`agent`** ticket marked `in-progress` in `state.json` with no commit behind it was interrupted mid-flight. Reset it to `pending` and run it again from scratch — a half-applied ticket is worse than a fresh one. A **`human`** ticket marked `in-progress` is not reset — the user may well have finished the work between sessions, outside the repo, where no commit would ever show it. Instead ask: „03 чакаше теб — вече готово ли е?“ and proceed from the answer exactly as `phases/5-subagents.md` describes for a human-ticket completion.
```

- [ ] **Step 2: Verify**

Run: `grep -n "human" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\0-preflight.md"`
Expected: one hit, inside the replaced sentence.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/0-preflight.md
git commit -m "Фаза 0: resume не ресетва human-тикети, пита дали вече е готово"
```

---

### Task 5: Phase 6 (Review) — state the human-ticket exemption

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\6-review.md`

**Interfaces:**
- Consumes: the "skip Phase 6 entirely" reference from Task 3, Step 2.
- Produces: nothing downstream.

- [ ] **Step 1: Add the exemption as the opening line of the phase**

Find the very top of the file:

```markdown
# Phase 6 — Checklist

Review of each ticket's diff along three axes. Not sequential — it runs inside Phase 5, after every ticket.
```

Replace with:

```markdown
# Phase 6 — Checklist

Review of each ticket's diff along three axes. Not sequential — it runs inside Phase 5, after every ticket. **Does not run at all for a `human`-owned ticket** — there is no diff, the user did the work outside the repo. If the user later reports a problem with something they marked done, treat it as a new finding through the normal manifest/briefing path, not a failed review: see `phases/5-subagents.md`, "When a ticket is `human`-owned".
```

- [ ] **Step 2: Verify**

Run: `grep -n "human" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\6-review.md"`
Expected: one hit.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/6-review.md
git commit -m "Фаза 6: изрично прескача human-тикети, няма diff за преглед"
```

---

### Task 6: Phase 2 (Briefing) — manual-mode requirement confirmation pass

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\2-briefing.md`

**Interfaces:**
- Consumes: nothing from earlier tasks — independent of the owner feature, bundled into this plan because the same design doc and the same user request cover both.
- Produces: nothing downstream (leaf task).

- [ ] **Step 1: Extend the "Manual mode" section**

Find:

```markdown
## Manual mode

Same rules, no cap. Keep asking until nothing blocking remains, then say so plainly: „Няма повече въпроси, пиша спецификацията“.
```

Replace with:

```markdown
## Manual mode

Same rules, no cap. Keep asking until nothing blocking remains, then say so plainly: „Няма повече въпроси, пиша спецификацията“.

**Manual mode also runs a confirmation pass over the manifest itself**, after the blocking questions and before moving to Phase 3. For every requirement — not just the ones a blocking question already touched — say, one at a time: „R03 съм го разбрал като «X» — правилно ли е?“ A correction is recorded into the manifest row verbatim, the same as any other briefing answer; a plain confirmation needs no manifest edit. This exists because manual-mode users are choosing to stay this involved — `full` and `semi` do not run this pass, and stay exactly as documented above.

Every manifest edit made anywhere in this phase — from a blocking answer, a fork, or this confirmation pass — is mirrored into `state.json`'s `requirements.items` (see `phases/7-instruments.md`) in the same edit, not batched for later. The dashboard should never be behind the manifest file by more than one write.
```

- [ ] **Step 2: Verify**

Run: `grep -n "confirmation pass" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\phases\2-briefing.md"`
Expected: one hit.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add skills/autopilot/phases/2-briefing.md
git commit -m "Фаза 2: ръчен режим потвърждава всяко изискване, не само блокиращите"
```

---

### Task 7: `SKILL.md` — one-line pointer to the new capability

**Files:**
- Modify: `C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\SKILL.md`

**Interfaces:**
- Consumes: nothing (documentation-only, pointer to Tasks 2–3's detail).
- Produces: nothing downstream.

- [ ] **Step 1: Add a line to the Phase table's "Produces" column for Plan/Subagents**

Find:

```markdown
| 4 Plan | `phases/4-plan.md` | `tickets/NN-*.md` (or none — see tiers) |
| 5 Subagents | `phases/5-subagents.md` | code, commits, `interfaces.md` |
```

Replace with:

```markdown
| 4 Plan | `phases/4-plan.md` | `tickets/NN-*.md` (or none — see tiers), each with an `owner` |
| 5 Subagents | `phases/5-subagents.md` | code, commits, `interfaces.md` — or, for `human`-owned tickets, a wait for the user's word |
```

- [ ] **Step 2: Verify**

Run: `grep -n "owner" "C:\Users\user\PhpstormProjects\autopilot-bg\skills\autopilot\SKILL.md"`
Expected: two hits, both inside the table row just edited.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git add SKILL.md
git commit -m "SKILL.md: споменава owner в таблицата на фазите"
```

---

### Task 8: Sync to the installed skill + end-to-end dogfood run

**Files:**
- Modify (copy, not edit): `C:\Users\user\.claude\skills\autopilot\phases\0-preflight.md`, `2-briefing.md`, `4-plan.md`, `5-subagents.md`, `6-review.md`, `7-instruments.md`, `dashboard-template.html`, and `C:\Users\user\.claude\skills\autopilot\SKILL.md`

**Interfaces:**
- Consumes: every file touched in Tasks 1–7.
- Produces: a working, live-in-this-session copy of the skill, verified end to end.

- [ ] **Step 1: Copy every changed file into the installed location**

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
cp SKILL.md "$HOME/.claude/skills/autopilot/SKILL.md"
cp skills/autopilot/phases/0-preflight.md "$HOME/.claude/skills/autopilot/phases/0-preflight.md"
cp skills/autopilot/phases/2-briefing.md "$HOME/.claude/skills/autopilot/phases/2-briefing.md"
cp skills/autopilot/phases/4-plan.md "$HOME/.claude/skills/autopilot/phases/4-plan.md"
cp skills/autopilot/phases/5-subagents.md "$HOME/.claude/skills/autopilot/phases/5-subagents.md"
cp skills/autopilot/phases/6-review.md "$HOME/.claude/skills/autopilot/phases/6-review.md"
cp skills/autopilot/phases/7-instruments.md "$HOME/.claude/skills/autopilot/phases/7-instruments.md"
cp skills/autopilot/phases/dashboard-template.html "$HOME/.claude/skills/autopilot/phases/dashboard-template.html"
```

- [ ] **Step 2: Confirm the copies match the source exactly**

```bash
diff SKILL.md "$HOME/.claude/skills/autopilot/SKILL.md"
diff skills/autopilot/phases/0-preflight.md "$HOME/.claude/skills/autopilot/phases/0-preflight.md"
diff skills/autopilot/phases/2-briefing.md "$HOME/.claude/skills/autopilot/phases/2-briefing.md"
diff skills/autopilot/phases/4-plan.md "$HOME/.claude/skills/autopilot/phases/4-plan.md"
diff skills/autopilot/phases/5-subagents.md "$HOME/.claude/skills/autopilot/phases/5-subagents.md"
diff skills/autopilot/phases/6-review.md "$HOME/.claude/skills/autopilot/phases/6-review.md"
diff skills/autopilot/phases/7-instruments.md "$HOME/.claude/skills/autopilot/phases/7-instruments.md"
diff skills/autopilot/phases/dashboard-template.html "$HOME/.claude/skills/autopilot/phases/dashboard-template.html"
```

Expected: every `diff` prints nothing (identical files). Any output means a copy step in Step 1 was skipped or failed — re-run it.

- [ ] **Step 3: Dogfood — regression check on a plain autonomous run**

This is the check that the change is truly additive. Reuse the existing finished T0 test project instead of building a new one:

```bash
node -e "
const fs = require('fs');
const p = 'C:/Users/user/PhpstormProjects/autopilot-test-cafe/.autopilot/state.json';
const s = JSON.parse(fs.readFileSync(p, 'utf8'));
console.log('tickets:', JSON.stringify(s.tickets));
console.log('has owner on any ticket:', s.tickets.some(t => 'owner' in t));
"
```

Expected: `tickets: []` (that project was T0, no tickets — confirms the field's absence is fine) and `has owner on any ticket: false`. Then regenerate its dashboard from the **updated** template and confirm it still renders with no console errors:

```bash
cp "$HOME/.claude/skills/autopilot/phases/dashboard-template.html" /tmp/cafe-regression-check.html
```

Manually diff the `<script>` portion's rendering logic against the live `C:\Users\user\PhpstormProjects\autopilot-test-cafe\.autopilot\dashboard.html` state block by opening `/tmp/cafe-regression-check.html` after pasting in that project's actual `STATE` object (between the markers), and confirm in a browser: stages, requirements list, and the "Готово" T0 note all render exactly as they did before this change (no owner badges appear anywhere, since no ticket declares one — `t0block` has no tickets to badge in the first place).

- [ ] **Step 4: Dogfood — one real mixed-owner ticket, end to end**

In the already-running `vipozi.com` project (`C:\Users\user\PhpstormProjects\vipozi.com`), once Phase 4 is reached with at least one ticket of each owner, this step is naturally exercised by the actual project work — no separate synthetic project is needed. Record here, for whoever implements this plan, what "passing" looks like when that happens:

- The plan shown to Alexander for his „ок“ names each ticket's owner.
- A `human` ticket, once launched, shows 👤 and "Чака теб" on the dashboard, with a running clock.
- Saying "готово" in chat for that ticket moves it to `done` on the dashboard within one state write, with no commit/test fields populated and no error.
- An `agent` ticket in the same wave runs concurrently, unaffected.

If Task 8 is being executed before the vipozi.com project reaches Phase 4, mark this step's checkbox once the synthetic render in Step 3 of Task 1 has been re-confirmed against the now-fully-copied installed skill (i.e. re-run Task 1 Step 8's sample-state render using the **installed** copy at `$HOME/.claude/skills/autopilot/phases/dashboard-template.html` instead of the repo copy, to prove the installed copy — not just the repo copy — renders correctly).

- [ ] **Step 5: Final commit (sync only — no doc changes in this repo)**

The installed-skill copy at `~/.claude/skills/autopilot/` is not a git repository tracked by `autopilot-bg` — nothing to commit there. Confirm the `autopilot-bg` repo's own log shows all seven prior commits from Tasks 1–7:

```bash
cd "C:/Users/user/PhpstormProjects/autopilot-bg"
git log --oneline -8
```

Expected: the 7 commits from Tasks 1–7 (plus whatever commit preceded this plan). Nothing left uncommitted:

```bash
git status --short
```

Expected: empty output.
