# Phase 5 — Crew

Where the code gets written. **Identical in all three modes — this phase is always hands-free.** Manual mode buys the user control over *what* gets built, not over each edit; once the tickets are agreed, the crew flies to the end without further approvals.

At tier T0 there are no tickets: you are the crew, working straight from the spec in the current context. Everything below about contracts and returns still applies to you — write `interfaces.md`, run the Phase 6 checklist, commit once.

**T0 does not excuse empty instruments.** Mark the `build` stage `active` before you start and `done` when you finish, record the pass in `state.json` under `singlePass` (files, tests, commit, both timestamps), and update the `requirements` counts exactly as a ticket would. A run that finished the whole project and left the user a dashboard showing nothing but a running clock has failed at the one job the dashboard has. See `phases/7-instruments.md`.

## One ticket, one subagent, one fresh context

Never two tickets in one context. Accumulated context is precisely what makes long vibecoding sessions start breaking things that used to work — the model stops reading and starts remembering, and its memory is worse than the files.

The corollary is that a subagent knows **nothing** except what you hand it. Hand it the right things.

## What a subagent gets

| | |
|---|---|
| `interfaces.md` | what previous tickets actually built — read in full, first |
| its ticket file path and body | including the verbatim brief quotes |
| the spec sections its ticket names | not the whole spec |
| the test command and how to run one file | so it does not have to derive them |
| the working directory and stack constraints | including what it must not touch |
| variable **names** for any credential | never a value, ever |

## interfaces.md — the shared contract

The file that keeps eight independent contexts building one coherent project instead of eight incompatible halves. Without it, ticket 06 invents a second version of what ticket 03 already built, and nobody notices until the end.

Created empty in Phase 0. **You** — the orchestrator — append to it after each ticket returns, from that ticket's contract block. Subagents never write to it: parallel writers would collide, and a subagent cannot know what the others produced.

```markdown
# Какво вече е построено

Чете се от всеки изпълнител преди началото на работата. Не изобретявай отново това, което вече го има тук.

## Общи правила на проекта

- Стек и версии, команди за стартиране и тестове
- Какво е забранено да се променя (конфигурационен файл, схема, общ модул и негов собственик)
- Ако липсва зависимост — не я добавяй сам, върни `BLOCKED` с името ѝ

## От подзадача 01 — скелет на проекта

- `db.connect(path) -> Connection` — единствената точка за връзка
- Таблици `requests`, `clients`; миграции в `migrations/`, собственик — подзадача 01
- Тестове: `npm test`, един файл — `npm test -- <path>`

## От подзадача 02 — приемане на заявки

- `createRequest({phone, address, problem}) -> {id, createdAt}`
- Валидация на телефон — `validatePhone(raw) -> {ok, normalized}`, не пиши своя
```

Keep it to interfaces and rules. It is not a log — the log is `state.json`.

## The return contract

Every subagent ends by returning exactly this. Put it in the prompt as a requirement, not a suggestion: without it you cannot update the instruments or the manifest, and the next ticket flies blind.

```
STATUS: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
FILES: създадени и променени
TESTS: команда → резултат (например `npm test` → 34 passed)
INTERFACES: публични сигнатури, схеми, формати на събития, които изложи навън
            — това, което следващите подзадачи ще използват
REQUIREMENTS: R01 done | R01.1 placeholder — <какво липсваше>
CONCERNS: какво е направено с уговорка и защо
BLOCKERS: какво липсваше (зависимост, решение, достъп)
```

`NEEDS_CONTEXT` means the ticket was under-specified — the executor could not tell what was wanted. Treat it as a defect in Phase 4, not in the executor: re-cut the ticket with the missing detail and run it again. Two `NEEDS_CONTEXT` in one flight means the tickets are too thin across the board — go back and merge.

## Order of flight — waves, not one at a time

Phase 4 left every ticket with a `wave` and a `zone`. Fly wave by wave, and inside a wave fly everything at once: **launch the whole wave in a single message, one subagent call per ticket.**

That last sentence is the whole section. Two subagent calls sent in two messages run one after the other — the parallelism was computed in the plan and then quietly thrown away in the delivery. This is the default failure, not a rare one: a serial flight looks exactly like a correct one from the inside, and the only visible symptom is a user waiting an hour for work that took twenty minutes of real dependency.

- **Cap at three in flight.** Beyond that the orchestrator's own context fills with returns it cannot usefully hold, and the whole point of the design leaks away. A wave of five goes out as three, then two.
- **Zones must be disjoint.** Phase 4 guarantees it within a wave; check again at launch, because a re-cut ticket may have moved into someone else's files. Overlap → the second one waits for the next slot. **Same files → serialise, no exceptions.**
- **A wave is not a barrier.** The moment one ticket returns, process it and launch the next ticket whose blockers are all done — even while its wave-mates are still flying. Waiting for the slowest ticket of a wave gives back exactly what the wave bought.
- **Nothing parallelises with ticket 01.** The shell, the schema, the shared primitives: everything else reads what it built.
- **When in doubt, serialise.** A wrong guess about disjoint files costs silent lost work; a serial run costs minutes.
- **In manual mode the flight is still hands-free.** Waves change how the agreed tickets are ordered, never which tickets get built.

## Before each ticket

Set the ticket's `status` to `in-progress` and its `startedAt` to now **before** launching the subagent, and mirror it into the dashboard. It costs one edit, and it is the difference between the user watching a ticket run and the user watching nothing happen for eighteen minutes.

For a wave, that is **one state write for the whole wave**, before the launch message — all of its tickets flipped together. Two clocks running side by side on the dashboard is what parallel work looks like; two tickets marked `in-progress` an edit apart is the same thing and costs half as much.

**A `human`-owned ticket does not get a subagent.** Mark it `in-progress` and `startedAt` exactly like an `agent` ticket — the clock still matters, the user should see how long they have been waiting on themselves — but do not launch anything for it. Instead, in the same message where you report the wave's launch, add one line naming what the user needs to do: „Готов си за 03 — настрой навигационното меню в Divi 5 Theme Builder по спецификацията. Кажи ми като приключиш.“ A wave with both kinds in it launches its `agent` tickets normally and announces its `human` tickets in the same breath — neither waits for the other.

## After each ticket

In this order, every time:

1. **Read the contract block.** No block → the ticket is not finished; ask the subagent for it.
2. **Append to `interfaces.md`.**
3. **Update the manifest** — `in-ticket` → `done` or `placeholder`, commit noted.
4. **Run the Phase 6 checklist** over the diff (`phases/6-review.md`).
5. **Run the full test suite**, not just the ticket's own tests. A regression introduced now costs minutes; found eight tickets later it costs the evening. Red → fix before moving on.
6. **Commit** — one commit per ticket, the ticket number in the subject. These are the user's rollback points.
7. **Update the instruments** (`phases/7-instruments.md`) — one line of state, one line of the dashboard: the ticket's `finishedAt`, tests and commit, the `requirements` counts, the `build` and `review` stage notes („3 от 5 подзадачи готови“), `updatedAt`.
8. **Top up the project memory — only if something was discovered.** The real test command, a gotcha that cost time, a new variable in `.env.example`. One line appended between the markers, never a rewrite; the architecture is written once, at the end. Most tickets add nothing, and that is the correct rate. Rules in `phases/9-memory.md`.
9. **Tell the user one plain-language line**: „Ботът приема заявки — 3 от 8 готови“. No diffs, no jargon, no file lists.

### When a ticket is `human`-owned

There is no contract block, no diff, no test run — the user did the work outside the repo. When they tell you it is done (by ticket id, by name, or unambiguously — "готово" when exactly one human ticket is waiting):

1. **Update the manifest** — the ticket's requirements move to `done`, Основание notes it was done by the user, no commit reference.
2. **Skip Phase 6 entirely.** There is nothing to review — see `phases/6-review.md`.
3. **Update the instruments** — `status: "done"`, `finishedAt` now, leave `files`/`tests`/`commit` as they were (empty). The dashboard already renders these as `—` when absent; that is correct here, not a gap to fill.
4. **Tell the user one plain-language line**, same as an agent ticket: „Менюто е готово — 3 от 8 подзадачи.“
5. If the user instead says it is **not** done, or only partly — do not mark it done. Ask what is missing and leave it `in-progress`. This is not a `failed` ticket (nothing failed); it is simply still open. Only mark `failed` if the user explicitly abandons the ticket, and treat that the same as any dropped requirement: back to the manifest, a quote, a question — never a silent status flip.

A `human` ticket never produces a `D##` row or an `interfaces.md` entry on its own — those exist for what code exposes to other code. If the user's manual work changes a decision the plan assumed (e.g. a menu structure that turned out not to fit the theme), that is still a plan-contradiction per "When the build contradicts the plan" below, handled the same way regardless of which kind of ticket surfaced it.

## When the user takes over a ticket already running

Allowed at any point, the same way a mode or depth switch is allowed mid-run: the user says something like „дай на мен тази задача“ for a ticket currently `in-progress` with `owner: "agent"`.

1. If a subagent is in flight for it, let it finish its current step rather than killing it mid-write — an interrupted code edit is worse than a finished one you then discard.
2. Set `owner` to `"human"` on that ticket.
3. Whatever the subagent already produced stays — this is a handoff, not a rollback. Tell the user in one line what exists so far and what is left: „Досега е готова схемата на менюто във файла; остава да я пренесеш в Theme Builder.“
4. From here it follows the `human`-ticket path exactly: no further subagent launches for it, marked `done` when the user says so, no Phase 6 review.

The reverse direction — a `human` ticket the user asks the agent to take over instead — follows the same shape: flip `owner` to `"agent"`, then launch a subagent for it exactly as Phase 5 would for any newly-launchable ticket, with the user's own description of what is already done included in its prompt so it does not redo finished work.

### When two tickets return together

Process them **one at a time, each through the whole list above**. Two returns are not one event.

- **One commit per ticket, always.** A shared commit takes away a rollback point the user paid for, and blames two tickets for one regression.
- **Run the full suite after each**, not once after both. Otherwise a red test has two possible authors and you have to bisect what you could simply have known.
- **`interfaces.md` is appended by you, in return order**, one block per ticket. Subagents never write to it — parallel writers collide.
- **Two returns claiming the same interface is a plan defect, not a merge problem.** It means the zones overlapped: keep the one that fits `interfaces.md`, and re-cut the other rather than reconciling two versions of the same thing by hand.

## When a ticket fails

Retry **once**, in a fresh context, with the error attached and the failing test named. If that fails too, one further attempt is allowed **only with a changed approach** — a different design decision, a different library, a path the ticket now names explicitly. Running the same attempt again with more hope is not a retry, and it is the only version of this that is forbidden.

After that the flight stops: tell the user in plain language what is blocking and what you need from them. Do not improvise around a blocker, and do not silently narrow the ticket to whatever happened to work — a quietly reduced ticket is a lost requirement, and this whole design exists to make that impossible.

Mark it `failed` in `state.json` and `placeholder` in the manifest, with the reason.

One failure does not abort its wave-mates — they are independent by construction, so let them land. What it does stop is everything **downstream**: its dependents stay `pending`, and naming which ones are now blocked is part of the sentence you tell the user.

## When the build contradicts the plan

The plan was written before the code existed, so sometimes the code is right and the plan is wrong: a data model that does not hold, an interface the spec assumed cannot exist, two requirements that turn out to be incompatible in practice. This is ordinary, it is not the executor's error, and it needs a path — because without one what actually happens is worse. The executor quietly builds something else, the spec keeps claiming otherwise, and every check downstream measures the build against a document that stopped being true at ticket four.

A subagent that hits this returns `BLOCKED` or `DONE_WITH_CONCERNS` with what it found. **You decide, in the orchestrator's context — never the executor**, and never by letting it stand:

1. **Amend the spec section.** Edit the affected part of `spec.md` in place, keep the story marks, and add one line saying what the code proved and at which ticket. From the first ticket onward the spec is a living document; the brief and the manifest quotes are not.
2. **Record a `D##` row in the manifest** — *discovered*. Its Основание is the finding, and it names the requirement it serves. This is not a requirement the user made; it is a constraint reality imposed, and it carries a status and appears in the final report like everything else.
3. **Re-cut only what the change invalidates.** Landed tickets stay landed. Unstarted tickets get their spec references updated; a ticket whose whole point disappeared is cut and its requirements go back to `in-spec` to be re-covered.
4. **Tell the user one line, in every mode including full:** „Схемата от плана не издържаше два адреса на една заявка — оправих го, изискването е същото“. They do not need the reasoning. They do need to know the plan moved, because a plan that moves silently is how the final report and their memory of the project stop matching.

Two things this is not:

- **Not a way to drop a requirement.** A requirement the code proves impossible is a question for the user — in full mode, an ASSUMPTION plus a placeholder — never a `D##` that quietly retires it.
- **Not a route for good ideas.** A discovery is something the code demonstrated, not something you thought of while writing it. Ideas are still `A##`, still need a parent and the proportion limit, and at `strict` are still forbidden.

## Testing

Test at the seams the spec named, not everywhere. Write the test before the code that satisfies it, one behaviour at a time — test, implementation, next. Tests written in bulk up front verify imagined behaviour and go numb to real changes.

A test asserts through the public interface and stays green through a refactor. If it breaks when the internals move but the behaviour did not, it is testing the wrong thing. And an expected value must come from somewhere other than the code under test — a known-good literal, a worked example, the spec. An assertion that recomputes the answer the way the code does can never disagree with it.
