# TASK_BOARD — операционная доска задач (истина по статусам)

> Протокол работы со строками: `docs/status/AGENT_PROTOCOL.md`. Статусы: `TODO | IN_PROGRESS | PARTIAL | BLOCKED | DONE | DEFERRED`.
> Поля строки: **ID · Задача · Объём/файлы · Критерий готовности (DoD) · Тесты · Зависит · Статус** (+Агент, +Начата/Готово).
> `IN_PROGRESS` обязан иметь `Агент` и `Начата`. `PARTIAL` обязан иметь `Сделано/Осталось/Точка продолжения`. `DONE` — дату и хэш в HANDOFF_LOG (в таблице: `DONE <дата>`).
> Новые задачи: только новый ID в конец своей фазы (дефекты гейта: `T-P{N}-FIX-{NN}`). Существующие ID не переиспользовать.
> Оценка: 1 задача ≈ 1 сессия агента. Задачи без «Зависит» могут стартовать немедленно.

---

## P0 — Фундамент (1/9)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P0-01 | Документационная система и протоколы | `AGENTS.md`, `docs/**` | Все спеки/роадмап/доска/промпты созданы и самосогласованы | ручная ревизия | — | **DONE 2026-09-07** (arena/01a07db6) |
| T-P0-02 | Скаффолд монорепо | `pnpm-workspace.yaml`, root `package.json` (скрипты dev/build/test/lint/typecheck/e2e), `tsconfig.base.json` strict, `eslint.config.js`, `.prettierrc`, `.nvmrc` (Node 22), обновить root README на структуру | `pnpm i` чисто; `pnpm -w typecheck` и `lint` проходят на пустых пакетах; стилистические правила применяются | смоук: CI-локально | — | TODO |
| T-P0-03 | packages/shared init | `packages/shared/*` | пакет собирается, zod зависимость, экспорт barrel, vitest конфиг с thresholds | 1 smoke-тест зелёный | T-P0-02 | TODO |
| T-P0-04 | apps/client init | `apps/client/*`: Vite+TS, `phaser@^4.2.1`, game config WEBGL 420×900, Boot/Preload-минимум, тёмный экран + текст-заглушка через i18n-stub | `pnpm dev` поднимает 5173 на 0.0.0.0, страница рендерит Phaser-канвас | ручной визуал + console log WebGL OK | T-P0-02 | TODO |
| T-P0-05 | apps/server init | `apps/server/*`: Fastify 5, pino, Zod-env плагин, `/health` → `{status,version}` | `pnpm dev` поднимает 8787 на 0.0.0.0; curl /health 200 | vitest inject-тест /health | T-P0-02 | TODO |
| T-P0-06 | CI пайплайн | `.github/workflows/ci.yml` | install(typecheck→lint→test→build) матрица отключена(одна ось), pnpm кэш; артефакты отчётов | зелёный прогон на ветке | T-P0-02 | TODO |
| T-P0-07 | Спайк rexUI × Phaser 4 | `apps/client/src/lab/rex-spike.ts`, `docs/adr/ADR-001-rexui-phaser4.md` | проверены: rexUI под Phaser 4 (2+ компонента рендерятся и интерактивны); зафиксировано решение: use / fork / внутренний ui-kit; ADR принят | в спайк-странице интерактивные Tabs+Slider | T-P0-04 | TODO |
| T-P0-08 | Зеркало master-материалов | `docs/reference/MASTER_INDEX.md` | перечень 9 файлов master с sha1 (git ls-tree), команда извлечения, краткие аннотации; скрипт `tools/sync-master.sh` | скрипт выводит список | — | TODO |
| T-P0-09 | Playwright каркас | `tests/playwright.config.ts`, devices-матрица, webServer поднимает клиент+сервер | `boot.spec`: канвас появляется, тёмный фон (#05070F), console без ошибок | e2e зелёный в CI | T-P0-04, T-P0-05, T-P0-06 | TODO |

## P1 — Shared-ядро (0/10)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P1-01 | Контракт: рынок | `shared/src/contracts/market.ts` | Candle/VolumeBar/LevelLine/MarketSlice/DatasetRef по SPEC 02 §1 | parse round-trip, отрицательные кейсы | T-P0-03 | TODO |
| T-P1-02 | Контракт: вкладки | `contracts/tabs.ts` | 10 TabKind дискриминированных payload | каждая kind минимум 1 фикстура | T-P1-01 | TODO |
| T-P1-03 | Контракт: сценарий | `contracts/scenario.ts` | ScenarioPublic/ScenarioSecret; Public не сериализует secret | **антилик**: глубокий обход — `fore|salt|scoringRubric|acceptableDecisions|realSymbol` отсутствуют в Public JSON | T-P1-02 | TODO |
| T-P1-04 | Контракт: скоринг/attempt | `contracts/scoring.ts`, `contracts/attempt.ts` | ScoringRubric/ScoreBreakdown/AttemptSubmit/Result/RevealPackage; hash-lock утилиты (canonical JSON + sha256 + salt) | hash known-answer тест; Σвесов=100 валидатор | T-P1-01 | TODO |
| T-P1-05 | Контракт: профиль/экономика/журнал/WS | `contracts/player.ts`, `contracts/ws.ts` | SPEC 02 §6–7 целиком, конверт WsEnvelope | parse всех типов сообщений | T-P1-01 | TODO |
| T-P1-06 | Фикстуры ×3 | `shared/fixtures/` + скрипт `fixtures:check` | сценарии: preEntry chart-only; inPosition + evidencePick; noise_quarantine протокол | все фикстуры валидны, используются в скоринге | T-P1-03, T-P1-04 | TODO |
| T-P1-07 | Движок скоринга v1 | `shared/src/scoring/*` | формула и веса SPEC-LOGIC §2; rules-DSLite матчер; golden-кейсы: «угаданный верх+harmful ≤30», «контролируемый выход 100», multi-acceptable частичные | branches 100%, snapshots | T-P1-04, T-P1-06 | TODO |
| T-P1-08 | Экономика/уровни/стрик | `shared/src/economy/*`, `shared/src/levels/*` | xpFromScore, награды, кэп фарма, computeLevel(матрица), streak «закрытая ошибка» | unit ≥95%, границы 0/99 | T-P1-05 | TODO |
| T-P1-09 | scenario-gen v1 | `packages/scenario-gen/*` вход {datasetSlice, contentDef, seed} → ScenarioPublic+Secret | детерминизм побайтовый; generatorVersion в выходе; `Math.random` отсутствует (lint) | snapshot ×3 seed; lint-правило | T-P1-03 | TODO |
| T-P1-10 | Селектор сценариев | `shared/src/selector.ts` | rule-based выбор по SPEC-LOGIC §10; вес паттернов +40%; тир = f(уровень) ±1 | детерминизм выбора; баланс-инварианты | T-P1-08, T-P1-06 | TODO |

## P2 — Сервер MVP (0/9)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P2-01 | Drizzle-схема и миграции | `apps/server/drizzle/*`, `db/client.ts` | таблицы SPEC 01 §7; миграции up/down; `db:seed:test` (контент из fixtures) | миграция-тест свежей БД | T-P0-05, T-P1-01 | TODO |
| T-P2-02 | Гостевая авторизация | `plugins/auth.ts`, `routes/auth.ts` | POST /api/auth/guest → {playerId, token}; защищённые маршруты 401 без токена | integration | T-P2-01 | TODO |
| T-P2-03 | Выдача миссий | `routes/missions.ts`, `services/missions.ts` | GET next (selector, seed с сервера), GET package (только Public, обфускация по SPEC 01 §9) | интеграция + антилик по сети | T-P2-02, T-P1-10 | TODO |
| T-P2-04 | Попытка и reveal | `routes/attempts.ts`, `services/scoring.adapter.ts` | POST attempts (идемпотентен по clientAttemptId) → score (сервером!) + revealToken; GET reveal отдаёт fore+salt, проверка foreHash; повторный reveal без новой попытки — 409/по политике | интеграция всей цепочки; гонка: 2 параллельных submit | T-P2-03, T-P1-07 | TODO |
| T-P2-05 | WS-шлюз | `ws/index.ts` | конверты SPEC 02 §7, ping/pong, переподключение не дублирует попытку | ws integration ×3 типа | T-P2-04 | TODO |
| T-P2-06 | Профиль и журнал API | `routes/profile.ts`, `routes/journal.ts` | GET profile (xp/level/streak/patternCounts), GET journal (стр., фильтры) | интеграция + пустые стейты | T-P2-04, T-P1-08 | TODO |
| T-P2-07 | Охрана и наблюдаемость | плагины rate-limit, security headers, request-id | конфиг-драйв; лимиты на attempts/reveal | тест 429 | T-P2-02 | TODO |
| T-P2-08 | Офлайн-sync приёмник | `routes/sync.ts` | POST /api/sync/outbox пакетом; idempotent; конфликт версий контента → 4xx с payload | интеграция: 3 записи, повтор | T-P2-04 | TODO |
| T-P2-09 | Документирование API | `apps/server/ROUTES.md` + curl-чеклист честности | каждый маршрут: пример запроса/ответа; чеклист hash-lock прогнан вручную | — | T-P2-04 | TODO |

## P3 — Клиентский каркас (0/22)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P3-01 | Дизайн-токены | `packages/content/design-tokens.json`, theme loader | токены SPEC 06 полные; читаются клиентом; ни один hex вне токенов в новых файлах | токен-аудит скрипт | T-P0-04 | TODO |
| T-P3-02 | UI-слой по ADR-001 | `game/ui/rexui-theme.ts` или `packages/ui-kit/*` (fallback) | Panel, TabPages, Slider, ScrollablePanel, Toast, Badge работают на демо-сцене | e2e демо-сцены | T-P0-07 | TODO |
| T-P3-03 | SceneRouter + оверлеи | `game/scenes/router.ts`, `overlay/*` | page-сцены sleep/wake; модалки/тосты поверх; пресеты переходов | unit-доступность API; e2e навигация Hub→Settings | T-P3-01 | TODO |
| T-P3-04 | Сторы Zustand | `stores/*` SPEC 01 §4 | все 6 сторов, persist+миграции, нет импорта Phaser | unit: actions + миграция v0→v1 | T-P0-03, T-P0-04 | TODO |
| T-P3-05 | EventBus bridge | `bridge/EventBus.ts` | store→scene события подписка/отписка без утечек | unit: отписка при shutdown сцены | T-P3-04 | TODO |
| T-P3-06 | netClient (mock) | `game/net/netClient.ts` | интерфейсы mission/attempt/reveal; mock-режим из fixtures; флаг server|mock | контрактные: payload парсится zod | T-P1-03, T-P3-04 | TODO |
| T-P3-07 | CandleChart v1 | `game/chart/CandleChart.ts` | grid/свечи/объёмы/ось цен/level-линии; пул текстур; dirty-render | unit draw-расчёты; perf ≤2мс; visual snap | T-P3-01 | TODO |
| T-P3-08 | CandleChart v2 | тот же | decision marker + fog + revealTo твин + skip; reduce-motion ветка | visual duo-state; unit reveal-index | T-P3-07 | TODO |
| T-P3-09 | Рендеры вкладок A | `ui/browser-widget/tabs/*` | chart, volumeLiquidity (стакан+китовая стена+funding/OI), news (карточки таблоида), sentiment (шкала) | visual snap ×4; zod payload parse | T-P3-02, T-P1-02 | TODO |
| T-P3-10 | Рендеры вкладок B | то же | higherTf (мини-чарт), macro, tokenData, onchain, position, journalSelf | visual snap ×6 | T-P3-09 | TODO |
| T-P3-11 | Рамка браузера | `ui/browser-widget/BrowserWidget.ts` | хром, адресная строка arena://, панель иконок 1–7, активный/новый стейт; «мёртвый ⟳» с подколом | e2e: переключение вкладок | T-P3-09 | TODO |
| T-P3-12 | CardTray | `ui/card-tray/*` | 4 состояния карты, звёзды мастерства, long-tap боттмшит, «cила 0%» — никаких бой-эффектов | e2e: select/locked тост; visual | T-P3-02 | TODO |
| T-P3-13 | DecisionBar | `ui/decision-bar/*` | 4 кнопки 2×2, цвета квадрантов, commit flow + блокировка | e2e: тап → commit, повторный тап невозможен | T-P3-02 | TODO |
| T-P3-14 | ExtraStepDock | `ui/extra-step/*` | 5 видов dock (riskSize slider с тиками, чипы ×3 вида, evidencePick мод), скрыт до решения | unit каждого; e2e в миссии | T-P3-13 | TODO |
| T-P3-15 | MissionScene композиция | `scenes/MissionScene.ts` | полная сборка SPEC 03 §4 из fixtures: topbar/ситуация/браузер/карты/решения/extra/entity-presence (3 tier-режима) | e2e mission.spec (mock); visual | T-P3-08, T-P3-11, T-P3-12, T-P3-14 | TODO |
| T-P3-16 | RevealScene | `scenes/RevealScene.ts` | play-forward, маркер «решение принято», разоблачение-печать, skip; ожидание результата | e2e часть mission; visual | T-P3-08 | TODO |
| T-P3-17 | DebriefScene | `scenes/DebriefScene.ts` | 3 слоя SPEC 03 §5, кольцо 0–100 с 5 полосами, rewards-чипы, «правило записано», CTA ×3 | e2e + visual ×3 вердикта | T-P3-05 | TODO |
| T-P3-18 | TopBar + BottomNav | `ui/top-bar.ts`, `ui/bottom-nav.ts` | ресурсы/уровень/звук; навигация MVP 4 пункта, активный neon | e2e переходы | T-P3-03 | TODO |
| T-P3-19 | HubScene lite | `scenes/HubScene.ts` | баннер ENTER BATTLE, мини-профиль, 3 слота дейли (пустышки по контракту), панk-фон ≤30% | visual | T-P3-18 | TODO |
| T-P3-20 | SoundManager | `game/audio/*` | шины master/music/sfx, 10 sfx по SPEC 06 §8, мьют, спрайт-пак | smoke: decode и play без ошибки | T-P0-04 | TODO |
| T-P3-21 | i18n | `src/i18n/*` | t(), ru.json+en.json, плюрали, fallback, все строки P3-экранов через ключи | i18n:audit чистый | T-P3-01 | TODO |
| T-P3-22 | PWA оболочка | vite-plugin-pwa конфиг, иконки-заглушки | manifest валиден (3 уровня иконок), SW ставится, офлайн-рестарт shell, update-prompt | pwa.spec ч.1 | T-P3-01 | TODO |

## P4 — Вертикальный срез MVP (0/16)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P4-01 | Связка клиент↔сервер | `netClient` server-режим | миссии/attempts/reveal ходят по REST+WS; mock-режим сохраняется флагом env | e2e mission.spec против сервера | T-P2-04, T-P3-15 | TODO |
| T-P4-02 | E2E честность | `tests/e2e/honesty.spec.ts` | fore отсутствует до commit (сетевая инспекция всего трафика), foreHash пересчёт совпадает, повторный reveal корректен | e2e зелёный | T-P4-01 | TODO |
| T-P4-03 | Гостевой вход | boot-flow, `sessionStore` | первый старт → guest token; рестарт сессии без потери профиля | e2e | T-P2-02, T-P4-01 | TODO |
| T-P4-04 | Onboarding 10 шагов | `scenes/OnboardingScene.ts` | SPEC 03 §3; пропуск объяснений; флаг в профиле; повтор из настроек | onboarding.spec | T-P4-01, T-P3-15 | TODO |
| T-P4-05 | Контент-пак MVP партия 1 | `packages/content/scenarios/*` ×15 | по матрице CONTENT §3; brief ≤140 зн; 3 протокола; валидатор P5-пока ручной по чек-листу | zod-прогон всех файлов | T-P1-06 | TODO |
| T-P4-06 | Контент-пак MVP партия 2 | ×10–25 | итог пула 25–40; inPosition ≥35%; каждая категория сущности ×3 | zod + coverage-скрипт (временный) | T-P4-05 | TODO |
| T-P4-07 | Карты 12: контент+иконки | `content/cards.json`, `assets-src/icons/cards/*` | пул LOGIC §9; принципы-однострочники в тоне; иконки центрально-объектные | визуальный чек 96px читаемости | T-P1-05, T-P3-12 | TODO |
| T-P4-08 | Сущности 8: контент+арт v0 | `content/entities.json`, арт v0 | пул LOGIC §8; weakness/counter/links заполнены; арт v0 допускает генеративные наброски по prompts | zod-прогон | T-P1-05, T-P3-11 | TODO |
| T-P4-09 | Академия 6 глав | `content/academy/*` | структура/риск/позиции/таймфреймы/ликвидность/психология; уроки ≤90с; связки карт/сценариев | e2e academy.spec | T-P1-05, T-P3-18 | TODO |
| T-P4-10 | Journal v1 | `scenes/JournalScene.ts` + API | история решений, patternCounts, «правило записано» создаёт запись | journal.spec | T-P2-06, T-P3-18 | TODO |
| T-P4-11 | Bestiary v1 | `scenes/BestiaryScene.ts` | список/фильтры/деталь/CTA «вызвать»; DEFEATED по закрытию темы | e2e сценарий разблокировки | T-P4-08, T-P3-18 | TODO |
| T-P4-12 | Профиль/Настройки lite | `scenes/SettingsScene.ts` | звук-шины, локаль RU/EN, reduce-motion, повторить onboarding, удалить данные | e2e смоук | T-P3-20, T-P3-21 | TODO |
| T-P4-13 | Офлайн outbox | netStore.outbox + sync ui | офлайн миссия → запись; сеть → sync; конфликт версии → понятный экран | offline.spec | T-P2-08, T-P4-01 | TODO |
| T-P4-14 | Стрик + награды HUD | playerStore + Debrief/TopBar анимации | «день с закрытой ошибкой»; чипы наград с твинами; кэп фарма отображается тостом | unit economy + visual | T-P1-08, T-P3-17 | TODO |
| T-P4-15 | Плейтест 90c | `docs/qa/PLAYTEST_P4.md` | 3 игрока, 10 случайных сценариев: медиана ≤90с; список UX-багов оформлен задачами FIX | чек-лист приложен | все предыдущие P4 | TODO |
| T-P4-16 | Закрытие MVP | bugfix sprint + GATE P4 | e2e-набор 1–7 и 10 зелёный; тег `v0.4.0-mvp`; GATE-запись с метриками | полный прогон | T-P4-15 | TODO |

## P5 — Контент-конвейер (0/12)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P5-01 | Датасет-загрузчик | `tools/fetch-klines.ts`, `datasets/registry.json` | загрузка klines выбранных пар/tf; sha256+license; политика хранения срезов задокументирована | воспроизводимость: 2 прогона → один хэш | P4 гейт | TODO |
| T-P5-02 | Валидатор контента v2 | `packages/content/validate/*`, `pnpm content:check` | все правила CONTENT §2 включая Σ100, 4 решения, fore≥12, i18n-ключи существуют; coverage-plan проверка | CI-шаг; тесты самого валидатора | T-P5-01 | TODO |
| T-P5-03 | Генеративная вариативность | scenario-gen v2 | орнаментализация (t0-окно, переформулировки brief, дистракторы) без троганья fore/рубрики | детерминизм + дифф-ревью выборки | T-P5-02 | TODO |
| T-P5-04 | AUTHORING.md | `packages/content/AUTHORING.md` | гайд автора: поля, обфускация, чек-лист «вкладка = влияет на решение», рубрико-дизайн | новый агент пишет сценарий ≤30 мин (замер в GATE) | T-P5-02 | TODO |
| T-P5-05 | EN-локаль + аудит | `locales/en.json` полный, `i18n:audit` CI | 100% ключей; аудит падает на захардкоженной строке | CI зелёный | T-P3-21 | TODO |
| T-P5-06 | Глоссарий панк-стиля | `content/copy/style-glossary.md` | словарик мемов/запретов + пак: пустые состояния, тосты, загрузки (каждый мем — по style-tone) | ревизия: шкала жестокости соблюдена | — | TODO |
| T-P5-07 | Превью-инструмент сценария | `apps/client/preview/` маршрут | JSON → рендер MissionScene без сервера; экспорт PNG-скриншота для PR | smoke e2e инструмента | P4 гейт | TODO |
| T-P5-08 | Контент до 150 сценариев | партиями по 25 | валидатор зелёный, покрытие матрицы ≥ плану, review чек-лист на партию | content:check + выборочная ручная игра | T-P5-02, T-P5-03 | TODO |
| T-P5-09 | Арт-пайплайн + сущности финал | `content/art/prompts.md`, атласы | воспроизводимые промпты; 8 стартовых сущностей — финальный арт в top1-стиле; атлас-сборка в CI | размер бандла атласов в бюджете | T-P4-08 | TODO |
| T-P5-10 | Иконки карт 12 финал + обложки глав | `assets-src/*` → атласы | читаемость 96px; единый ракурс рамки; обложки 6 глав | визуальный ревью-лист | T-P4-07, T-P5-09 | TODO |
| T-P5-11 | Фоны и заставки | панк-коллаж пак | фоны Hub/Debrief/турнир при ≤30% шума; 3 заставки загрузки | визуальный ревью-лист | T-P5-09 | TODO |
| T-P5-12 | Релиз-процесс контента | docs + CI | версия, ченджлог, PR-чеклист с визуальными снапами новых сценариев | демо: релиз контента 0.5.0 | T-P5-02 | TODO |

## P6 — Метагейм (0/10)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P6-01 | Уровни 0–99 UI | профиль: матрица устойчивости | computeLevel из shared; визуал «чего не хватает до уровня»; L99 недостижимость задокументирована | unit границы; visual | T-P1-08 | TODO |
| T-P6-02 | Дейли-квесты | quests модуль + Hub виджет | 3 типа (сыграть N / закрыть ошибку / урок); claim наград; сброс | e2e claim | P4 гейт | TODO |
| T-P6-03 | Streak freeze + локальные пуши | economy + notifications | freeze за монеты; opt-in пуши PWA; тексты в тоне | unit + ручной dev-пуш | T-P6-02 | TODO |
| T-P6-04 | Инвентарь и косметика-флаги | `scenes` + stores | equip темы чарта (первый реальный эффект косметики!), рамки карт, аватар; инвентарь работает на внутренних флагах, SKU-контракт подхватит T-P8-01 | e2e equip flow | P4 гейт | TODO |
| T-P6-05 | Pattern insights v2 | Journal + selector | рекомендации сценариев из паттернов; «закрытие паттерна» церемония (3 подряд без повтора) | unit церемонии; e2e журнал | T-P4-10, T-P1-10 | TODO |
| T-P6-06 | Экономический симулятор | `tools/econ-sim.ts` | 1000 бот-сессий: приток валют положителен при ошибках; кэп фарма работает; отчёт в docs | отчёт приложен к GATE P6 | T-P1-08 | TODO |
| T-P6-07 | Навигация 5 пунктов | bottom-nav реконфигурация | Академия/Бестиарий/Арена/Маркет(заглушка-«скоро»)/Турниры(заглушка) без сломанных e2e | e2e навигация обновлена | P4 гейт | TODO |
| T-P6-08 | Разблокировка сущностей главами | Academy↔Bestiary | закрытие главы → сущность «изучена» → DEFEATED-трекинг; уведомление-печать | e2e цепочка | T-P4-09, T-P4-11 | TODO |
| T-P6-09 | Адаптация v1.1 | selector | анти-серийность (≤2 подряд одного тира), усталость дистракторов, окно разнообразия сущностей | unit инварианты серий | T-P1-10 | TODO |
| T-P6-10 | Сезонный каркас | content/seasons | сезон 1: структура, даты, косметика-плейсхолдеры, «пасс-награды» данные без покупок | zod сезона | P5 гейт | TODO |

## P7 — Турниры / PvP (0/8)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P7-01 | Турнирные сущности | drizzle + content defs | tournament {id, window, scenarioSet frozen, rules}; seed-цепочка участника | unit генерации набора: 2 игрока — идентично | P2 гейт, T-P6-07 | TODO |
| T-P7-02 | Entry flow + окно | TournamentScene | запись, статусы окна, серия миссий в режиме протокола | e2e | T-P7-01 | TODO |
| T-P7-03 | Leaderboard + tie-break | services/leaderboard | каскад: качество → протокол → доказательства/extra → скорость; UI таблицы | unit каскада ×все связки | T-P7-02 | TODO |
| T-P7-04 | Reveal после окна | attempts в турнир-режиме | fore недоступен до конца окна (или конца своей попытки — политика в конфиге турнира), серверно | security-тест API | T-P7-02 | TODO |
| T-P7-05 | Анти-аномалии | services/integrity | флаги speedrun/идентичных паттернов; очередь manual review (простая таблица) | unit хьюристик | T-P7-03 | TODO |
| T-P7-06 | Share-картинка | canvas-генератор OG-карточки результата | рендер в памяти → share/dl; панк-стиль | visual snap | T-P7-03 | TODO |
| T-P7-07 | E2E турнир | tournament.spec | 2 аккаунта, полный цикл, tie-break сработал | зелёный | T-P7-04 | TODO |
| T-P7-08 | Duels (опционально) | feature flag `duels` | вызов по ссылке, тот же набор | e2e за флагом | T-P7-07 | TODO |

## P8 — Маркет и монетизация (0/8)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P8-01 | SKU и entitlement | content/sku.json + shared | каталог косметики/паков/пасса/подписки; **контрактный запрет**: схема SKU не содержит полей влияния (score/risk/data/лимиты) | контракт-тест «нет полей силы» | P6 гейт | TODO |
| T-P8-02 | MarketScene полный | вкладки Магазин/Пасс/Подписка/Паки | по UX §11 + микро-дислеймер на каждой карточке; концепт-2 тон | visual + e2e покупка-stub | T-P8-01 | TODO |
| T-P8-03 | IAP-абстракция | `services/payments/*` | интерфейс провайдера (stub + web), валидация чека на сервере, entitlement → inventory; идемпотентность | integration: повторная доставка чека | T-P8-02 | TODO |
| T-P8-04 | Сезонный пасс | MarketScene пасс-вкладка | 30 уровней наград из T-P6-10; покупка stub; прогресс от XP | e2e | T-P8-03, T-P6-10 | TODO |
| T-P8-05 | PRO-подписка флаги | entitlements | без рекламы(её нет — флаг на будущее), расширенная аналитика журнала, ранний доступ к новым наборам; **никакого влияния на скоринг** | paywall.spec | T-P8-03 | TODO |
| T-P8-06 | Честность покупок | test suite + UI текст | paywall.spec зелёный; текст «что вы покупаете / чего не покупаете» до оплаты | e2e | T-P8-03 | TODO |
| T-P8-07 | Цены и регионы | config + тесты | таблица цен ($5–15 сезон ориентир), округления, отображение до оплаты | конфиг-тесты | T-P8-01 | TODO |
| T-P8-08 | Equip-flow витрины | inventory UI связка | купленное надевается за 2 тапа; тема чарта визуально меняется | e2e | T-P6-04, T-P8-03 | TODO |

## P9 — Полировка и релиз (0/10)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P9-01 | Перф-аудит | бюджеты 07 §6 в CI | bundle ≤1.6МБ br, fps-пробник зелёный, redraw чарта ≤2мс | CI перф шаг | P4 гейт | TODO |
| T-P9-02 | A11y проход | все экраны | контрасты ≥4.5, тапы ≥44, reduce-motion везде, фокус-порядок (web-обвязка) | чек-лист + axe на html-оболочке | P6 гейт | TODO |
| T-P9-03 | Аналитика событий | shared events + server sink | D1/сессия/миссии/воронка onboarding/retake-rate; first-party, без PII, opt-out | контракт событий + интеграция | P4 гейт | TODO |
| T-P9-04 | Crash/error reporting | клиент+сервер | self-hosted вариант или отключено по env; scrub PII | инъекция ошибки → событие | T-P9-03 | TODO |
| T-P9-05 | Правовые страницы | контент + экран | «не финансовый совет», приватность, условия покупок — RU/EN, человеческим языком (не панком) | ручная ревизия юристом отмечена | T-P5-05 | TODO |
| T-P9-06 | Стор-листинг | assets + тексты | скриншоты ×5, иконки 3 уровней по бренду, описания RU/EN в тоне (с правовой врезкой) | ревью-чеклист | T-P5-11 | TODO |
| T-P9-07 | Release CI | workflows/release.yml | тегованые сборки, changelog из коммитов, артефакты, rollback-инструкция | dry-run релиза | T-P0-06 | TODO |
| T-P9-08 | Нагрузочный прогон | k6/artillery скрипт | 100 rps 10 мин на key API без 5xx; отчёт | GATE-артефакт | P2 гейт | TODO |
| T-P9-09 | Софт-лаунч | 10 игроков, чек-лист | D1 замерен на когорте; triage багов в FIX-задачи | отчёт | все P9 выше | TODO |
| T-P9-10 | RELEASE 1.0 | тег, постмортем | GATE P9 подписан целиком; `v1.0.0`; постмортем в docs/qa | RELEASE-запись | T-P9-09 | TODO |

## P10 — Пост-релиз (0/4)

| ID | Задача | Объём/файлы | DoD | Тесты | Зависит | Статус |
|----|--------|-------------|-----|-------|---------|--------|
| T-P10-01 | Токен/GameFi RFC | `docs/adr/RFC-token.md` | юрид-архитектура, токеномика, «работает при цене 0», комплаенс-аудит план; **без кода** | ревизия заинтересованных | RELEASE 1.0 + PMF доказан | TODO |
| T-P10-02 | Creator marketplace RFC | RFC + прототип-спека | авторские пакеты, ревью-процесс, доля; анти-sybil | — | RELEASE 1.0 | TODO |
| T-P10-03 | Локализация 10+ | процесс + party | приоритетные языки по аналитике; community-переводы процесс | аудит ключей всех локалей | T-P5-05, RELEASE 1.0 | TODO |
| T-P10-04 | Эффективность обучения | исследование | до/после-метрики навыков (anonymized), публикация результатов | — | аналитика P9 + 3 мес | TODO |

---

### Шаблон новой задачи (копировать в конец фазы)

```md
| T-P{N}-{NN} | <имя> | <файлы/модули> | <DoD — проверяемо> | <тесты> | <T-…> | TODO |
```

### Шаблон PARTIAL-строки (в статус-колонку)

`PARTIAL — Сделано: … | Осталось: … | Точка продолжения: <файл:место + следующий шаг>`
