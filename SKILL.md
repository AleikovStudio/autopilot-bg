---
name: autopilot
description: Use when the user dictates an app, site, bot, or feature to build end-to-end and expects a finished result without reviewing specs, tickets, or code — vibecoding sessions, non-technical users, "направи го до ключ", "build it for me", "не ме питай излишно" requests. Also use when the user invokes /autopilot, or asks for a build in a named mode or depth — «пълен автомат», «ръчен режим», «строго по брифа», «помисли задълбочено».
argument-hint: "[full|semi|manual] [strict|deep] какво да се направи или път до brief.md"
---

# Autopilot

## Overview

Autopilot flies a dictated idea from words to a working project **in one dialogue**, without making the user approve each stage. It is self-contained: every rule it needs lives in `phases/`. No other skill has to be installed.

Two ideas carry the whole design.

**The order is the product.** Code is written in the second-to-last phase. Everything before it exists to decide *what* to build, and everything after it exists to prove the right thing got built.

**The brief is the contract, not the design.** Two obligations follow from it, and they pull in opposite directions on purpose.

*Nothing may quietly vanish.* The user's original words become a numbered manifest before anything else happens, and every phase is gated on it. What breaks naive vibecoding is not bad code — it is a requirement that stopped existing somewhere around the third rewrite.

*The brief is not the design.* It is a silhouette: it describes the happy path and nothing underneath — no empty states, no failures, no interruptions, no limits. Working those out is legitimate work, not scope creep, and it is where much of the value of this process comes from. **How far to take it is the user's dial**, set by the [depth](#depth) parameter. What is never allowed at any setting is depth that **detaches** from the brief.

## Reading this skill

This file is the orchestrator: modes, phase order, gates. The rules for each phase live in `phases/` and are **read at the moment that phase starts, not before** — that is what keeps the working context small.

| Phase | Read | Produces |
|---|---|---|
| 0 Preflight | `phases/0-preflight.md` | repo configured, `.autopilot/` created |
| 1 Manifest | `phases/1-manifest.md` | `brief.md`, `manifest.md` |
| 2 Briefing | `phases/2-briefing.md` | answers recorded into the manifest |
| 3 Spec | `phases/3-spec.md` | `spec.md` |
| 4 Plan | `phases/4-plan.md` | `tickets/NN-*.md` (or none — see tiers) |
| 5 Subagents | `phases/5-subagents.md` | code, commits, `interfaces.md` |
| 6 Review | `phases/6-review.md` | per-ticket review |
| 7 Instruments | `phases/7-instruments.md` | `state.json`, `dashboard.html` (opened for the user) |
| 8 Final | `phases/8-final.md` | blind acceptance, final report |
| 9 Memory | `phases/9-memory.md` | `CLAUDE.md` / `AGENTS.md` — the project as the next session will find it |

## The words the user sees

The phases have English names in this file and the user never sees them. In the chat, on the dashboard and in the final report there is **exactly one Bulgarian word per stage**, and it is this one. Two vocabularies for one process is how a person reads the README and then cannot find any of it on the screen.

| Phase | `stages[].id` | За потребителя |
|---|---|---|
| 0 Preflight | `preflight` | Подготовка |
| 1 Manifest | `manifest` | Изисквания |
| 2 Briefing | `briefing` | Брифинг |
| 3 Spec | `spec` | Спецификация |
| 4 Plan | `plan` | План |
| 5 Subagents | `build` | Разработка |
| 6 Review | `review` | Код-ревю |
| 8 Final | `final` | Приемане |

Two rules hold this together:

- **„Сглобяването“ е целият процес, а не един етап.** „Сглобяването върви“, „сглобяването прекъсна“, „продължи сглобяването“ — това е за целия проход. Затова петият етап се казва „Разработка“: иначе една дума ще значи и частта, и цялото. А „билд“ в смисъл на `npm run build` — също не е той.
- **Единицата работа е „таск“.** Не „задача“, не „тикет“, не „issue“. „Задача“ е това, което е поставил потребителят (брифът); една дума за две различни неща обърква и отчета, и таблото.

Phases 7 and 9 are not sequential. The instruments are written and opened in Phase 0, then updated at every stage transition and after every ticket. The project memory is raised in Phase 0, topped up when the build discovers something, and written in full in Phase 8 by a subagent reading the finished code.

## Modes

Everything typed after `/autopilot` splits into three parts: **the mode** (optional bare word — `full`, `semi`, `manual`), **the depth** (optional bare word — `strict`, `deep`), and **the brief** (everything else). No dashes on either parameter. Text that is not a recognised parameter is always brief.

`/autopilot full deep онлайн магазин за керамика` — full mode, deep elaboration. Order does not matter; both parameters are optional and independent.

| Mode | Triggers | Human gates |
|---|---|---|
| **full** — пълен автомат | `/autopilot full`, „пълен автомат“, „пълен автопилот“, „изцяло сам“, „не ме питай нищо“, "fully automatic", "don't ask me anything" | none |
| **semi** — полуавтомат **(default)** | `/autopilot semi`, „полуавтомат“, nothing specified | questions only |
| **manual** — ръчен | `/autopilot manual`, „ръчен режим“, „одобрявай всяка стъпка“, "ask me everything", "approve every step" | questions + spec + tickets |

- **Announce the resolved mode and offer the others, once, before Phase 1.** The user must never discover the mode by noticing questions that did or did not arrive — and they cannot ask for a mode they do not know exists. In a chat client there is no `--help` to read: this block is the only place the dials are ever named, so it is not optional.

  ```
  Режим: полуавтомат · дълбочина: обичайна — ще питам само това, което не е ясно от задачата, останалото поемам аз.
  Отворих таблото: .autopilot/dashboard.html — обновява се само.
  Памет за проекта — AGENTS.md (+ CLAUDE.md с връзка към него). Кажи, ако искаш друг файл.

  Може да превключиш по всяко време, просто кажи:
  • „пълен автомат“ — не питам нищо
  • „ръчен режим“ — одобряваш спецификацията и списъка с таскове
  • „строго по брифа“ / „помисли задълбочено“ — по-малко или повече домисляне отвъд казаното
  ```

  One short block, once, at the start. **It is a hint, not a question** — say it and go straight into Phase 1; waiting for a reply to it is exactly the pause this skill exists to remove. Do not repeat it later, do not restate it after a mid-run switch (one line is enough there: „Разбрах, оттук нататък — ръчен режим“).
- **Ambiguity resolves to semi.** A mode word contradicting the rest of the sentence („ръчен режим, но не питай“) → the explicit mode word wins; two mode words → ask which one, in one line.
- **The mode can be switched mid-run** („превключи на ръчен“) — it applies from the next phase onward. Phases already passed are not replayed.
- **Extra instructions in the brief** (stack, language, budget, „без база данни“, deadline) are manifest requirements like any other. They constrain the build; they never replace a phase.
- **No mode removes the manifest gates or the safety gates.** Irreversible or outward-facing actions — deploy, publish, pay, send messages to third parties, delete data, rewrite git history — stay a question in **all three** modes, including full.

## Depth

How far past the brief's own words the spec is allowed to go. The mode decides *how much the user is asked*; depth decides *how much is worked out for them*. They are independent.

| Depth | Triggers | Deepening a requirement (`R##.n`) | New capabilities (`A##`) |
|---|---|---|---|
| **strict** | `/autopilot strict`, „строго по брифа“, „само това, което казах“, „не добавяй нищо“, "strictly as written", "nothing extra" | only what the requirement cannot work without | **not allowed** |
| **normal** **(default)** | nothing specified | freely, by judgement — as much as the feature warrants | allowed, with a parent, within proportion |
| **deep** | `/autopilot deep`, „помисли задълбочено“, „максимална дълбочина“, „помисли вместо мен“, "go deep", "think it through" | the full depth pass, every dimension, every requirement | actively encouraged, same two limits |

- **Default is normal, and normal means permitted.** The agent elaborates where elaboration obviously helps and does not chase every edge of every requirement. This is the setting most briefs should run on.
- **`strict` does not mean careless.** Errors and empty states are still handled — a build that crashes on bad input does not satisfy the requirement it was written for. What `strict` removes is anything the user did not ask for: no extra capabilities, no anticipating needs, no „докато съм тук, да добавя и това“.
- **`deep` does not lift the attachment rules.** Every `A##` still names its parent requirement; the proportion limit still holds. `deep` buys thoroughness, never a different project.
- **Depth is announced with the mode**, in the same opening block: „Режим: полуавтомат · дълбочина: максимална“.
- **Depth can be changed mid-run** („по-малко самодейност“, „помисли по-задълбочено“) — applies from the next phase. Already-written spec sections are not retroactively trimmed unless the user asks.

The rules for each level live in `phases/3-spec.md`.

## When to Use

- User dictates what to build and expects the finished thing, not a collaboration on process.
- User is non-technical: will not read specs, judge ticket granularity, or review code.
- „Направи го до ключ“, „just build it“, „не ме питай излишно“.
- User wants to approve the spec and the tickets but not to run the pipeline by hand — that is **manual** mode, still Autopilot.

**When NOT to use:** the user wants to co-author the code itself line by line (work with them directly); the task is a small single-file change (just do it); the idea is bigger than one project and its destination is unclear (settle the destination first, then return here).

## The flight

| Phase | full | semi (default) | manual |
|---|---|---|---|
| 0 Preflight | auto | auto | auto |
| 1 Manifest | auto | auto | auto |
| 2 Briefing | skipped → self-briefing | only what the brief leaves open — sometimes none | the same, with more patience |
| 3 Spec | auto | auto | show → wait for explicit „ок“ |
| 4 Plan | auto, notify only | auto, stoppable | discuss → wait for explicit „ок“ |
| 5 Subagents | auto | auto | auto |
| 6 Review | auto | auto | auto |
| 8 Final | report + Assumptions | report | report |

**The manifest gates run in every mode.** They are checks against the user's own words, not requests for the user's time — no mode buys the right to skip them.

| Gate | After phase | Condition to pass |
|---|---|---|
| **G1** | 2 Briefing | every requirement has a status; none left `open` without a reason recorded |
| **G2** | 3 Spec | every live requirement is `in-spec`, `deferred`, or `dropped`. Zero `open` |
| **G3** | 4 Plan | every `in-spec` maps to ≥1 ticket, **and every ticket traces back to ≥1 requirement** |
| **G4** | 8 Final | blind acceptance run; every disagreement with the manifest reported |

A failed gate is not a warning. It sends the phase back to be redone — see `phases/1-manifest.md`.

**The plan may be corrected; the brief may not.** When the build proves the plan wrong — a data model that does not hold, an assumed interface that cannot exist — the spec is amended and a `D##` row records what the code demonstrated and when. That is the one thing allowed into the manifest after the briefing, it never retires a requirement, and it is never a route for an idea you had. Rules in `phases/5-subagents.md`.

## Secrets

Credentials are the user's to hold, not the agent's to handle. This section binds every phase; the phases do not restate it.

- **Never request one.** No key, token, password, connection string, or card number is ever a question. *Which* provider is a question. *Whether* an account exists is a question. The credential is not.
- **Redact at ingest, before anything is written.** The brief, every user answer, and every pasted fragment pass the redaction gate in `phases/1-manifest.md` *before* they reach a file. A detected secret becomes `[REDACTED:<VAR_NAME>]` — the variable name survives, the value does not.
- **"Verbatim" always means "verbatim after redaction."** Wherever this skill asks for the user's exact words, it asks for them redacted. The two rules are one rule.
- **Refer to it by name.** `STRIPE_SECRET_KEY`, not the value. The user puts the value in `.env` themselves; `.env` is in `.gitignore` before the first commit; the final report lists which names are still empty.
- **A leaked secret is a stop condition.** A secret that reached a file or a commit is reported immediately, in plain language, with the advice to rotate it. Before the first commit, run the redaction gate over the whole of `.autopilot/`.

## Files this skill owns

```
.autopilot/
├── <feature-slug>/
│   ├── <YYYY-MM-DD>-brief.md   the user's original words, redacted, never edited again
│   ├── manifest.md      R01…Rnn — requirements and their status
│   ├── spec.md          the specification
│   ├── interfaces.md    what finished tickets built, for the tickets that follow
│   └── tickets/NN-<slug>.md
├── README.md            how to read this folder — for the human, written once in Phase 0
├── state.json           machine-readable run state: stages, tickets, timings, debt
└── dashboard.html       the human view — opened automatically in Phase 0, refreshes itself

CLAUDE.md | AGENTS.md   the project memory — what the next session reads first
```

The brief is dated in its filename because a slug directory outlives one sitting. The dashboard is opened for the user, not described to them: it shows the eight stages of the cycle, where the run is now, and a live clock on the run, the current stage and the current ticket.

`.autopilot/` is the record of **this** run; the memory file at the root is the project as it stands, for whoever opens the repo next. Autopilot's content there lives between `<!-- autopilot:start -->` markers — everything the user wrote outside them is untouchable. See `phases/9-memory.md`.

`.autopilot/` is committed, not ignored — it is the user's record of what was promised and what was delivered. A run that leaves nothing under `.autopilot/` did not happen.

## Judgement

This skill describes a process, not the product. Its numbers — tiers, question counts, story counts, wave widths — are **calibration for a first guess, never targets to hit.** A spec written to reach a story count, or a plan cut to land inside a tier, has optimised for the rule instead of for the person who asked.

The rules below are the same kind of thing. Each one is here because it was paid for, and each is an argument — arguments can lose. Where following one would make the result worse for the user, break it deliberately, say so in one line, and carry on. That is a decision, and decisions get recorded. What is never acceptable is breaking one quietly, or keeping one because it is written down.

**Four rules are not calibration and do not lose.** They hold in every mode, at every depth, at every tier:

1. **A requirement is removed only by the user**, in their own words, quoted into the manifest.
2. **A secret is never requested, echoed, or written** — not into a file, a prompt, a commit, or a report.
3. **A fact about the user is never invented.** Prices, texts, addresses, accounts stay visible placeholders until they supply them.
4. **An irreversible or outward-facing action is a question** — deploy, publish, pay, message a third party, delete data, rewrite history.

Everything else is argument.

## Rationalizations — the ones that cost the user the product

Phase-specific mechanics are not here; they live in the phase that owns them. What follows is the short list of excuses that end with the user getting something other than what they asked for.

| Excuse | Reality |
|--------|---------|
| „Потребителят каза да не задавам въпроси“ | Каза да не задавам ИЗЛИШНИ. Решаващите въпроси са част от работата, а не обсъждане на процеса. |
| „KISS — просто го направи“ | Простият резултат идва от ред, а не от прескачане на етапи. Без спецификация всяка поправка е „ама аз имах предвид друго“. |
| „Брифът е целият в диалога, защо да го преписвам във файл“ | Диалогът се свива, а брифът в него е най-старото нещо. След три фази ще съчиняваш по преразказ на преразказа. |
| „Това изискване явно не е важно, ще го пропусна“ | Кое е важно решава потребителят. Ти можеш да предложиш `deferred` — да го зачеркне може само той. |
| „Потребителят повече не го спомена — значи е отпаднало“ | Мълчанието не отменя нищо. Отмяната са негови думи, записани в манифеста като цитат. |
| „Ще сложа заместител, после ще уточни“ | Блокиращите неизвестни (плащане, хостинг, акаунти) се решават в брифинга — при пълен автомат чрез самобрифинг — но винаги преди разработката. |
| „Нека прати ключа, аз ще го сложа в кода“ | Ключовете ги въвежда потребителят и то само в `.env`. Ти работиш с името на променливата. |
| „Ключът вече е в контекста, значи мога да го запиша“ | Точно обратното: значи трябва да го заличиш и да предупредиш. Контекстът не е разрешение. |
| „По-бързо е всичко в един контекст“ | По-бързо е през първия час. После моделът обикаля в кръг и чупи вече работещото. |
| „Брифът е кратък — значи и спецификацията е кратка“ | Брифът е силует: потребителят е описал happy path и не е описал нито празни състояния, нито грешки, нито прекъсвания. При обичайна и максимална дълбочина да ги обмислиш е твоя работа. |
| „Това е очевидно, няма да го пиша“ | Очевидното за теб не е записано никъде и всеки субагент ще си го домисли по своему: трима изпълнители — три различни „очевидности“. Манифестът и спецификацията са единствените общи точки. |
| „Измислих полезна функция, ще я добавя“ | Задълбочаване на поръчаното (`R##.n`) — да. Нова възможност (`A`) — само с родителско изискване, в рамките на пропорцията и вписана в отчета. При `strict` — изобщо не. |
| „Пълен автомат — значи мога и да деплойна“ | Автоматичният режим маха въпросите за продукта, но не дава право на необратимо действие. Деплой, плащане, разпращане, изтриване — гейт във всички режими. |
| „В пълен автомат мога да реша всичко вместо потребителя“ | Решенията — да, и всички отиват в ASSUMPTIONS. Фактите за потребителя (цени, текстове, акаунти) — не: заместител и ред в отчета. |
| „Ще напиша «стартирам след 60 секунди»“ | Ти не можеш да чакаш — обещаната пауза няма да се случи. Честната формулировка е: „започвам, кажи стоп“. |
| „В ръчен режим също ще започна и ще изчакам възражения“ | В ръчния режим съгласието е изрично „ок“. Мълчанието не е съгласие, а започнатата работа — още по-малко. |
| „Ще сверя резултата със спецификацията, това стига“ | Спецификацията може вече да е изгубила изискване. Финалната сверка минава по брифа и без спецификацията — иначе тя ще потвърди собствената си грешка. |
| „Тасковете и спецификацията се виждат в чата — защо са ми файлове“ | Файлът в `.autopilot/` е артефактът; чатът е само негов преразказ. Диалогът ще изчезне, файловете ще останат. |
| „Потребителят не е питал за режимите — няма да го натоварвам“ | Той и няма да попита: в чата няма `--help`. Петте реда в началото са единственото място, където изобщо научава, че има какво да се настройва. |
| „Проектът е готов, тестовете са зелени — значи работи“ | Тестовете ги е писал същият процес, който е писал и кода. Докато никой не е стартирал проекта, „работи“ е хипотеза — а първият, който ще го стартира, е потребителят. |

## Red Flags — start the phase over

Every line here means something the user asked for is at risk. Phase mechanics — instruments, timestamps, wave bookkeeping, memory-file detection — are checked in the phase files that own them, not here.

- Writing code before the spec exists.
- The brief was never written to its file — the run is anchored to nothing.
- A requirement left the manifest without a status, or was marked `dropped` without a quote of the user saying so.
- Past gate G3: a ticket that traces to no requirement, or a requirement that traces to no ticket.
- Spec or tickets that exist only in the dialogue — nothing written under `.autopilot/`.
- Instruments that disagree with the chat: a stage still `active` after you moved on, a ticket running while the dashboard calls it `pending`, a ticket carrying the run's `startedAt` instead of its own, timestamps filled in afterwards from memory. The user believes the screen over your sentences, which is the whole reason it exists.
- The announced depth and the actual spec diverge: a bare restatement of the brief at normal or deep, or an invented capability — any `A##` — at strict.
- Final acceptance measured against the spec instead of blind against the brief.
- The blind checker or the memory subagent handed `spec.md` or the tickets. Independence is the entire mechanism; without it both of them confirm the plan instead of the code.
- The finished project was never actually run — accepted on green tests and a reading of the code.
- Starting without announcing mode and depth, or announcing one and behaving as another: questions in full, a start-and-see instead of „ок“ in manual.
- A blocking unknown — payment, hosting, an account, where the data lives — left unasked in semi or manual because the brief „изглеждаше ясен“. Asking nothing is legitimate only when nothing is open; a manufactured question and a skipped blocking one are both defects, in opposite directions.
- Promising the user a wait — a countdown, „след минута“, „ако не отговориш до N секунди“ — that you have no way to honour.
- In full: an invented fact about the user standing where an ASSUMPTION, a stub, or a PLACEHOLDER belongs.
- Asking the user a process question — which tracker, which doc file, which memory file, ticket granularity, code review — outside manual, where spec and tickets are gates by design.
- A requirement quietly narrowed to whatever happened to work, or the spec amended mid-build with no `D##` row recording why.
- Two tickets in one subagent context, or two tickets in one commit.
- Parallel subagents editing the same files — or the mirror failure, independent tickets flown one at a time with the plan's parallelism thrown away in the delivery.
- A subagent launched without `interfaces.md`, or finishing without returning the contract block.
- Payment, hosting, or accounts first mentioned at the finish line.
- A secret value asked for, repeated back, or written into any file, prompt, commit, or report.
- Installing a package or fetching remote code without the user asking for it.
- Text outside the `autopilot` markers edited, moved or dropped, or the run ending with no project memory file at all.
