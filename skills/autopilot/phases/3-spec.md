# Phase 3 — Flightplan

Turn the manifest and the briefing answers into a specification. **This phase does not reopen the interview** — what is still unresolved becomes a `placeholder` row, not another round of questions.

One exception, and it is narrow: a genuine fork the briefing missed, where the two branches are different projects and a placeholder would only defer the same question to the build. Ask it, once, in one line, with a recommended answer. In **full** mode there is no exception — decide it and record the `ASSUMPTION`.

Write to `.autopilot/<slug>/spec.md`. What the user sees in the dialogue is a two-line summary; the file is the spec.

## Depth

**The brief is a silhouette.** The user describes what happens when everything goes right, at the level of „приема заявки и ги записва в таблица“. They do not describe what the third screen says when the network drops, what an empty list looks like, what happens on a double submit, or what the very first launch shows before any data exists. They are not withholding those answers — they do not have them, and they are not supposed to.

Working them out is the most valuable thing this phase can do. A spec that merely restates the brief in tidier words has produced nothing, and it guarantees the gaps get filled during implementation by whichever subagent hits them first, differently each time.

**How far to take it is the user's setting, not yours.** Read it from the announced depth:

| Depth | The depth pass below | `A##` new capabilities |
|---|---|---|
| **strict** | not run. Only *Wrong input* and *Failure*, and only where the requirement plainly breaks without them | **forbidden** — cut them, do not write them |
| **normal** *(default)* | run by judgement — the dimensions that plainly matter for that requirement, skipping the ones that do not | allowed, with parent and proportion |
| **deep** | run in full — every dimension, every requirement, or an explicit „не е приложимо“ | encouraged, same parent and proportion |

At **strict**, an idea you had that the brief did not ask for does not become a spec section and does not become a note. It is simply not written. The user chose this setting to get exactly what they asked for, and a spec that argues with that choice has ignored an instruction.

At **normal**, use judgement rather than a checklist. Elaborate where the gap would obviously cause a bad build; do not chase every edge of every requirement. Most briefs belong here.

At **deep**, the pass is mechanical and exhaustive — that is what was asked for.

### The depth pass

Run at **normal** and **deep** (see the table above for how completely). Every `R##` requirement goes through this list.

At **deep**, skipping a dimension because a requirement „и без това е ясна“ is exactly the mistake this pass exists to prevent — write „не е приложимо“ instead of skipping silently.

| Dimension | The question the brief never answered |
|---|---|
| **First run** | what does this look like before any data exists? |
| **Empty** | zero items, zero results, zero history — what is on screen, and what invites the next step? |
| **Wrong input** | what does the user see, in their language, and what survives of what they typed? |
| **Failure** | the network, the service, the disk — what breaks, what is said, what is retried, what is lost |
| **Interruption** | closed halfway, refreshed, sent twice — does it resume, restart, or duplicate? |
| **Growth** | ten items versus ten thousand — what has to change, and does it now or later? |
| **Boundaries** | who may do this, and what happens to someone who may not? |
| **Aftermath** | where does the result go, who learns about it, can it be undone? |

Each answer becomes an `R##.n` story with its own acceptance line. `R##.n` never counts against any limit — it is the requirement the user actually made, worked out properly.

**The number of stories is an output, not a target.** A requirement that genuinely has seven dimensions gets seven stories; one that has two gets two, and padding it to look thorough is the same defect as skipping it. What is always wrong is twelve requirements producing twelve stories — that is the brief copied out rather than worked through.

### Two roads for depth

**Propose it** — when the answer is genuinely the user's to give (which of two behaviours they want, what tone the messages take, whether something is worth its cost). That is a briefing question, and one of the best a briefing can spend itself on.

**Decide it** — when the answer is craft, not preference. Error text, retry policy, empty-state copy, sensible limits, sane defaults. Decide, write it into the spec, and say so plainly in the summary. Asking the user which HTTP status to return is a wasted question; asking whether a cancelled order should refund automatically is not.

In **full** mode both roads collapse into the second — decide everything, and list what you decided in the final report. At **strict** depth, the first road narrows too: a question exists to clarify what the user asked for, never to sell them something extra.

### Keeping depth attached

The one failure mode worth guarding: depth that **floats free of the brief** — a beautiful subsystem grows while a plain requirement from line 2 quietly never gets a section. So every story carries a mark saying where it came from.

| Mark | Origin | Rule |
|---|---|---|
| `R##` | straight from the brief | untouchable |
| `R##.n` | **deepening** a brief requirement | uncapped — this is the main work of the phase |
| `G##` | decided in the briefing | the user confirmed it |
| `A##` | a **new capability** the brief never implied | must name a parent `R##` |
| `D##` | a constraint the **build** proved, added mid-flight | only from `phases/5-subagents.md`, never from an idea |

Note the line between the last two, because it is the one that gets blurred: elaborating „приема заявки“ into retry, resume and validation is `R01.n` — the same requirement, understood properly. Adding a loyalty programme is `A`. Depth is not scope creep, and treating it as if it were is how specs end up thin.

Three rules keep additions attached. They hold at **every** depth — `deep` relaxes none of them, and `strict` removes `A##` entirely:

- **Parenthood.** An `A##` story names the `R##` it serves. A free-floating invention gets cut — not because inventions are bad, but because one with no parent is a different project.
- **Proportion.** `A` stories may not outnumber `R` + `G` combined. `R##.n` never counts toward this — deepening what was ordered is unlimited by construction.
- **Precedence.** Recorded here, enforced in Phase 4: tickets closing `R` come before tickets closing only `A`. If time or patience runs out, what goes unfinished is your addition, never the user's request.

Every `A##` that survives into the build is listed in the final report under „какво добавих извън поръчаното“, so the user learns about it from the report rather than from the code.

## The template

```markdown
# Спецификация: <име на проекта>

## Задача

Проблемът на потребителя с негови думи — от лицето на този, който страда
от него, не от лицето на кода.

## Решение

Какво ще се появи пред потребителя, когато всичко е готово. Пак без техника.

## Потребителски истории

| # | Етикет | История | Приемане |
|---|-------|---------|----------|
| 1 | R01 | Като клиент, оставям заявка на бота, за да не се обаждам | ботът прие и потвърди |
| 2 | R01.1 | …и виждам разбираема грешка, ако връзката е паднала | текст на грешката, не мълчание |
| 3 | A01 → R01 | …и получавам номер на заявката, за да мога да го цитирам | номерът е в потвърждението |

Всяка история е „Като <кой>, аз <какво>, за да <защо>“ плюс това,
по което личи, че работи.

## Решения по реализацията

Стек, модули и техните граници, схема на данните, API контракти, външни услуги.
Всяко решение — с един ред „защо така“.
Без пътища до файлове и без код: те остаряват по-рано от спецификацията.

Изключение: ако структурата (схема, тип, машина на състоянията) се изразява
по-точно с код, отколкото с проза — вмъкни само нея, без обвивката около нея.

## Точки за тестване

Къде се проверява поведението — през публичните граници, не през вътрешностите.
Предпочитай вече съществуващите точки пред нови. Колкото по-малко са, толкова по-добре;
идеалът е една. Назови ги изрично: Phase 5 тества само там.

## Извън обхвата

Какво съзнателно НЕ строим. Всеки ред сочи изискване от манифеста,
което отлага, и казва защо.

| Изискване | Защо не сега |
|---|---|
| R06i — администраторски панел за заявки | заявките се виждат в таблицата; отделен екран — следваща итерация |

## Незапълнени места

Всеки `placeholder` от манифеста: какво стои като заместител, къде точно в кода,
и какво е нужно от потребителя, за да се запълни.

## Покритие на манифеста

| Изискване | Раздел от спецификацията |
|---|---|
| R01 | Истории 1–3, Решения §2 |
| R02 | Истории 4–5, Решения §4 |

Точно толкова редове, колкото живи изисквания има в манифеста. Нито едно пропуснато.
```

## The vocabulary rule

If `CONTEXT.md` or `docs/adr/` exist, the spec speaks the project's language — the terms already defined there, not synonyms. A concept you need that the glossary lacks is a signal: either you are inventing language the project does not use, or there is a real gap worth noting. If a decision here contradicts a recorded one, say so out loud in the spec rather than overriding it silently.

## Gate G2 — before leaving this phase

Update every manifest row: `open` → `in-spec` with its section, or → `deferred` with its Out of Scope line.

Then check: **zero `open` rows.** An `open` row means the spec does not cover something the user asked for. That is not a note for later — it is an incomplete spec. Go back and write the missing section.

This is the single most valuable check in the whole flight. Everything downstream trusts the spec; this is the last moment the spec is still comparable to the words the user actually said.

## Showing it

**semi and full** — two lines in the chat: what will be built, and what deliberately will not. Then move on.

**manual** — the spec is a gate. Show it in full, stop, wait for an explicit „ок“. Rewrite on every objection and ask again. Silence is not agreement, and neither is work already started.
