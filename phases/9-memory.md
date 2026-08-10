# Phase 9 — Project memory

The file the **next** session reads. Not a phase in sequence — started in Phase 0, topped up during the build, finished in Phase 8.

Three files describe this project and they are not interchangeable. Confusing them is how documentation rots.

| File | Question it answers | Lifetime |
|---|---|---|
| `.autopilot/<slug>/` | what was promised and what was delivered **in this run** | forever, but it is history |
| `.autopilot/<slug>/interfaces.md` | what the previous tickets built, for the tickets still to come | **dies with the run** |
| `CLAUDE.md` / `AGENTS.md` | what an agent needs to work in this repo **tomorrow** | forever, and it is the present tense |

The last one is what this file is about. Everything in it must be true of the repository *as it stands* — not of the plan, not of the run that produced it.

## Phase 0 needs two things from this file

Read this block and stop; nothing else here applies until the build is over.

1. **Pick the file** — the detection table immediately below, first match wins. An existing file always beats detection, the choice is recorded in `state.json` as `memoryFile`, and it is never a question for the user in any mode.
2. **Write the skeleton** — the block under *Moment 1*, between the markers, with only what is already known. An invented command is worse than a missing one.

What may be appended during the build (*Moment 2*) and the full description written from the finished code (*Moment 3*) are read when they happen — the second inside Phase 5, the third in Phase 8.

## Which file

Decided by detection, in this order. Stop at the first match.

| Check | File |
|---|---|
| `CLAUDE.md` already exists | `CLAUDE.md` |
| `AGENTS.md` already exists | `AGENTS.md` |
| both exist | the one that already holds the project description; if neither does, `AGENTS.md` — and **leave the other file alone** |
| `.claude/` directory, or `$CLAUDECODE` / `$CLAUDE_CODE_ENTRYPOINT` is set | `CLAUDE.md` |
| `.cursor/` directory | `AGENTS.md` |
| `.codex/` directory, or `.github/copilot-instructions.md` | `AGENTS.md` |
| nothing matched | `AGENTS.md` as the real file **+ `CLAUDE.md` containing one line: `Виж @AGENTS.md`** |

- **An existing file always wins over detection.** The repo has already answered the question; asking it again is how you end up with two half-filled memory files.
- **The pointer file is written only in the fallback case.** When the agent was identified, one file is enough — a second file is a second thing to keep in sync, and it will not be kept in sync.
- **Never duplicate the text into both files.** Two copies of a project description drift within one run.
- Record the choice in `state.json` as `memoryFile`, so a resume does not re-derive it.

**This is never a question for the user** — in any mode, including manual. It is a process decision, like where ticket files live, and Phase 0 answers those itself. It is not, however, a *secret* decision: one line in the opening block, together with the mode.

> Памет за проекта — `AGENTS.md` (+ `CLAUDE.md` с връзка към него). Кажи, ако искаш друг.

Say it and move on. **Do not wait for an answer** — if the user names a different file later, switch and move the block; renaming a markdown file costs nothing, which is exactly why this never earned a gate.

## Where the content lives — the markers

Everything Autopilot writes sits between two markers, in every case, including a file it created itself:

```markdown
<!-- autopilot:start -->
...
<!-- autopilot:end -->
```

One rule, and it buys two things: updating is „replace what is between the markers“, and **anything the user wrote outside them is untouchable**. A brownfield repo whose CLAUDE.md carries a team's hard-won rules must come out of an Autopilot run with those rules intact.

If the markers are missing on a later run but Autopilot's sections are recognisably there, wrap them — do not append a second copy.

## Moment 1 — the skeleton (Phase 0)

Cheap, written before anything is built, and it is what survives an interrupted run. Only what is already known:

```markdown
<!-- autopilot:start -->
# <Име на проекта>

<Един ред: какво е това и за кого.>

## Команди

| Команда | Какво прави |
|---------|-------------|
| `<инсталация>` | Инсталира зависимостите |
| `<стартиране>` | Стартира локално |
| `<тестове>` | Пуска тестовете |

## Как работи Autopilot тук

Разработката се води от умението `/autopilot`. Изисквания, спецификация и таскове — в `.autopilot/`.
Прогрес — `.autopilot/dashboard.html`. Правило: изискване от `manifest.md`
може да махне само потребителят.

Ако работата продължава — кажи „продължи автопилота“: състоянието ще се вдигне
от `.autopilot/state.json`, не е нужно да питаш нищо отново.
<!-- autopilot:end -->
```

Commands that are not known yet are simply absent. **An invented command is worse than a missing one** — the next session runs it, it fails, and now the whole file is suspect.

## Moment 2 — during the build

Append only **facts that were discovered and cost something to discover**. One line each, no rewrite of the file:

- the real test command, once it is known — and how to run a single file;
- a gotcha that ate time: an ordering dependency, a version pin, a platform quirk;
- a new variable in `.env.example`;
- a decision a subagent had to make that the next one must not re-litigate.

That is the whole list. What must **not** go in, from the CLAUDE.md quality rules:

- generic advice („пиши тестове“, „използвай разбираеми имена“) — true everywhere, useful nowhere;
- restatements of the obvious („класът `UserService` работи с потребители“) — the name already said it;
- one-off fixes and commit-by-commit history — that is what `.autopilot/` and git are for;
- long explanations of a standard technology — a link or one clause, never a paragraph;
- anything that duplicates `interfaces.md` while the run is still going. Interfaces are folded in **once**, at the end.

If nothing was discovered during a ticket, nothing is written. Most tickets write nothing, and that is the correct rate.

## Moment 3 — the full description (Phase 8)

Now the code exists, so now the architecture can be described from the code instead of from the plan.

**Spawn a subagent.** It runs in parallel with the blind-acceptance agent — they read the same finished repo and never see each other's output.

It receives: the repository, the current memory file, `interfaces.md`, the tier, and the commands to run and test the project.

**It must not receive `spec.md` or the tickets.** A memory written from the spec documents intentions; the next session trusts it and gets lied to by a file whose whole job is to be trusted. Same reasoning as the blind acceptance — different purpose, identical mechanism.

Its brief:

> Опиши проекта така, че агент, който отваря това хранилище за първи път, да
> започне работа без разузнаване. Източник — само кодът, който виждаш.
>
> Пиши стегнато: по един ред на мисъл. Не преразказвай очевидното от имената,
> не давай общи съвети, не обяснявай какво представляват познатите технологии.
> Всяка команда трябва да се стартира с копи-пейст, всеки път — да съществува.
>
> Ако нещо го няма в кода — раздела го няма. Празен раздел е по-лош от липсващ.

### What the sections are, by tier

The file scales with the project, exactly like the ticket tiers do.

**T0–T1 — кратък файл:**

| Раздел | Какво съдържа |
|---|---|
| Заглавие и ред | какво е това и за кого |
| Команди | инсталация, стартиране, тестове, компилация — проверени |
| Структура | дърво от 5–15 реда, всяка папка — с предназначението си |
| Подводни камъни | неочевидното, което вече е ухапало някого |
| Как работи Autopilot тук | от скелета, без промени |

**T2–T3 — плюс към това:**

| Раздел | Какво съдържа |
|---|---|
| Ключови файлове | входни точки и модули, до които ще се стига най-често |
| Архитектура | как частите са свързани: поток на данните, кой кого вика, къде са границите |
| Конвенции на кода | приетите в този проект, не в света изобщо |
| Среда | имена на променливите и защо е нужна всяка — **никога стойности** |
| Тестове | с какво и как; къде стоят; как се пуска един файл |

### Folding in interfaces.md

`interfaces.md` is a working contract between tickets, and its life ends with the run. Its durable content — public signatures, schemas, event formats, module ownership — becomes the Архитектура and Ключови файлове sections. What does not survive: the per-ticket framing („От таск 03…“), anything already obvious from the code, and any instruction addressed to a subagent.

The file itself stays in `.autopilot/<slug>/` as the run's record. It is not deleted and it is not maintained.

### Before writing — verify

Currency is the criterion this file fails first and most quietly. So, before the block is written:

1. **Run the commands** it documents — at minimum install, test, and build. A command that fails does not go in.
2. **Check every path** exists.
3. **Grep the block for secret values** — the redaction gate from `phases/1-manifest.md` applies here as it does everywhere. Variable names, never values.
4. **Check the length against the tier.** A landing page with a two-page memory file has been padded, and padding is how a reader learns to skim.

Then write the block between the markers, commit it with the final commit, and note the chosen file in the Phase 8 report under „Къде какво стои“.

## On resume

The memory file is the **first** thing to read on resume, before `state.json` — it is the cheapest possible re-entry into a project. If it is missing or plainly stale against the code, that is a defect of the previous run: fix it as part of the current one, do not work around it.
