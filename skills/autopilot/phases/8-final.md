# Phase 8 — Touchdown

Landing. Two things happen here, and the first one is the reason this framework exists.

## 1. Blind acceptance — gate G4

Every check so far has measured the build against the **spec**. But the spec is your own paraphrase of the brief, written several phases ago. If a requirement was lost on the way into it, everything downstream has been faithfully confirming that loss.

So the last check does not use the spec.

**Spawn a subagent that receives:**

- `.autopilot/<slug>/<дата>-brief.md` — the user's own words (the path is `briefFile` in `state.json`)
- the repository as it now stands
- how to run the project and its tests

**It must not receive:** `spec.md`, `manifest.md`, the tickets, or any summary of them. A checker given the spec inherits the spec's blind spots and will confirm them. Independence is the entire mechanism — take it away and this phase is theatre.

Its brief:

> Прочети приложения файл с брифа — това е задачата, поставена от клиента. После проучи
> хранилището и определи какво от нея наистина е реализирано.
>
> **Стартирай проекта** — командите са в приложеното описание — и мини през основния сценарий така,
> както би минал през него клиентът. Четенето на кода показва намерение, стартирането показва резултат.
> Ако проектът не тръгва или сценарият прекъсва — това е главната находка
> на проверката, постави я на първо място. Ако изобщо не може да се стартира (нужен е акаунт,
> ключ, външна услуга) — кажи направо какво точно е попречило, и не представяй четенето на кода
> като проверка на работоспособността.
>
> За всяко изискване от брифа: реализирано / частично / не — и един ред,
> където точно личи това (какво видя при стартирането, или къде е в кода,
> или защо реши, че го няма).
>
> Не оценявай качеството на кода. Не предлагай подобрения. Не търси оправдания
> за липсата — просто отбележи факта. Ако изискването е изпълнено формално,
> но по същество не работи (данните се пазят, но не се показват на потребителя) —
> това е „частично“, а не „реализирано“.

**Then compare its verdict with `manifest.md`:**

| Manifest says | Blind says | Meaning |
|---|---|---|
| `done` | реализирано | agreed |
| `done` | **частично / не** | 🔴 **drift** — the manifest is wrong. Report it, and fix it if it is small |
| `placeholder` | частично | expected — confirm the placeholder is visible, not an invented fact |
| `dropped` / `deferred` | не | expected — must appear in the report as not built |
| — | реализирано, но не е от брифа | scope that grew without a parent; report it |

Every 🔴 goes in the report **and** in `state.json` under `blind`. A drift found here is not a failure of the run — it is the run working. Hiding it is the failure.

If there are no tickets (tier T0), this check still runs. Small builds drift too, and it is one subagent.

**A build that was never run is a build nobody has seen work.** The tests were written by the same process that wrote the code, so they agree with it by construction; the first time this project meets a user must not be the first time it is launched. If it genuinely cannot be run here — no credentials, a service that needs an account, a platform this machine is not — that goes in the report as an open item under „какво е нужно от теб“, not silently into the accepted column.

## 2. The project memory — written from the code

**Launch this at the same time as the blind acceptance.** Two subagents, the same finished repository, no contact between them: one asks „какво от брифа е направено“, the other asks „как ще се ползва това утре“. Running them in parallel costs one wall-clock slot instead of two.

The memory agent writes the full description of the project into `CLAUDE.md` or `AGENTS.md` — architecture, key files, conventions, environment, tests, gotchas — scaled to the tier, folding in what `interfaces.md` accumulated. Like the blind checker, **it does not receive `spec.md` or the tickets**: a memory written from the plan documents intentions, and the next session has no way to tell the difference.

Everything about it — which file, the markers, the sections per tier, what must never go in, and the verification pass over the commands — is in `phases/9-memory.md`. Read it before spawning.

This is the artifact that decides what the *next* run costs. A project whose second session begins by re-reading the whole codebase paid for that in the first session and got nothing.

## 3. The final report

Run the full test suite once more first, and wait for both subagents. Then write in the user's language, plain, no jargon.

Order matters — the user reads the top and skims the rest.

**In full mode, the report opens with „Решения, взети вместо теб“** — every `ASSUMPTION` from the self-briefing, in plain language, each with the one-line reason. They never asked for these; they have the right to see all of them in one place, first.

```markdown
## Готово

<Какво работи сега — 3–6 реда на обикновен език, от лицето на потребителя.>

**Стартиране:**
```
npm install && npm run dev
```
Отвори http://localhost:3000

## Какво е нужно от теб

1. Попълни в `.env` — `TELEGRAM_BOT_TOKEN`, `GOOGLE_SHEETS_ID`.
   Файлът `.env.example` вече стои до него — копирай го и попълни стойностите.
2. Замени заместителите: цените в `src/data/prices.ts`, текстовете на писмата
   в `src/emails/`. В момента там стоят видими маркери `[ПОПЪЛНИ]`, а не измислени стойности.

## Какво не влезе

| Какво | Защо |
|---|---|
| Известия по SMS | ти каза „SMS не е нужно, само телеграм“ |
| Администраторски панел за заявки | отложено: заявките се виждат в таблицата, отделен екран — следваща итерация |

## Какво добавих извън поръчаното

<Всяка `A##`-история, стигнала до кода — на обикновен език, с изискването,
заради което е добавена. Разделът се пропуска само ако няма добавки
(при дълбочина `strict` — винаги). Потребителят трябва да научи за тях оттук,
а не като се натъкне на тях в кода.>

| Какво съм добавил | Заради какво |
|---|---|
| Номер на заявката в потвърждението | за да може клиентът да се позове на нея — R01 |

## Какво не мина по плана

<Всеки ред `D##` от манифеста — на обикновен език: какво е било замислено,
какво е попречило и как е направено вместо него. Разделът се пропуска само ако
няма `D##`. Изискването остава същото — сменя се начинът, не поръчката.>

| Какво не проработи | Как е направено вместо това |
|---|---|
| Една заявка на един адрес — половината клиенти имат по два адреса | Адресите са изнесени в списък, формата приема няколко |

## Открити въпроси

<Разминавания от сляпото приемане, ако има. Направо, без смекчаване:
„Изискването «клиентът вижда статус» го смятах за готово, но независимата
проверка показа, че статусът се пази и никъде не се показва. Поправено /
изисква отделна подзадача.“>

## Къде какво стои

- Описание на проекта за следващия път — `AGENTS.md` в корена
- Прогрес и числа — `.autopilot/dashboard.html`
- Първоначалната ти задача — `.autopilot/<slug>/<дата>-brief.md`
- Изискванията и съдбата им — `.autopilot/<slug>/manifest.md`
- Спецификация — `.autopilot/<slug>/spec.md`
```

## Rules for the report

- **Заместителите и празните променливи са задължителен раздел**, дори когато са нула (тогава с един ред: „всичко е попълнено“). Това е, което отделя „работи“ от „работи при теб“.
- **Секретите — само с имена.** Никога със стойности, включително тези, които потребителят е изпратил сам.
- **„Какво не влезе“ се пише винаги**, дори когато всичко е влязло. Празен раздел с един ред е по-честен от липсващ: показва, че въпросът е бил зададен.
- **Не разкрасявай.** Паднал тест, недовършена подзадача, открито разминаване — назовават се направо, с това какво точно е счупено и какво е нужно за поправка. Отчет, който крие дефект, излиза по-скъпо от самия дефект.
- **Никакви дифове, имена на файлове с код, имена на тестове** — те са в инструментите, за тези, на които са нужни.

## Closing the instruments

The memory file goes in with the final commit, before this. Then: set `finishedAt`, write the `blind` block, refresh the counts in `state.json`, close every stage — `final` to `done`, and anything still `active` or `pending` to `done`, `skipped` (with a note) or `failed`, whichever is true — then mirror into `dashboard.html`. A run whose dashboard says „в процес“ a day after it landed is lying to the person who trusted it.

If the dashboard lives in an in-app pane, re-point the pane at it once here — this is the picture the user is left with.

`finishedAt` also stops the clocks and the ten-second self-refresh: the page freezes on the final numbers instead of counting time nobody is spending. Leave it `null` on a finished run and the user's total keeps growing overnight.
