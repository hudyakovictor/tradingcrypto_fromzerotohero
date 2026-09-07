# SPEC 01 — ARCHITECTURE

## 1. Монорепо (pnpm workspaces)

```
/
├── package.json            # root: scripts-оркестраторы (dev/test/lint/typecheck/e2e/build)
├── pnpm-workspace.yaml
├── tsconfig.base.json      # strict: true, noUncheckedIndexedAccess, exactOptionalPropertyTypes
├── eslint.config.js        # flat config, ts + import-order; правило-no-math-random (см. §8)
├── AGENTS.md  docs/  …
├── apps/
│   ├── client/             # Phaser 4 PWA (порт 5173 dev)
│   │   ├── index.html  public/  vite.config.ts  playwright/
│   │   └── src/
│   │       ├── main.ts            # new Phaser.Game(config)
│   │       ├── game/
│   │       │   ├── config.ts      # WebGL, Scale.FIT, 420×900 design space, parent, fps
│   │       │   ├── scenes/        # Boot, Preload, Hub, Mission, Reveal, Debrief,
│   │       │   │                  # Academy, Bestiary, Journal, Market, Tournament, Settings
│   │       │   ├── ui/            # browser-widget/ card-tray/ decision-bar/ top-bar/
│   │       │   │                  # extra-step/ bottom-nav/ modal/ toast/ rexui-theme.ts
│   │       │   ├── chart/         # CandleChart.ts (см. §5)
│   │       │   ├── fx/            # particles, tweens-пресеты, juice
│   │       │   ├── audio/         # SoundManager-обёртка, sfx-ключи
│   │       │   └── net/           # netClient: REST + WS адаптеры, offline-resolver
│   │       ├── stores/            # zustand vanilla сторы (см. §4)
│   │       ├── bridge/            # EventBus: stores ⇄ scenes
│   │       └── i18n/              # t(key), ru.json/en.json (плоские ключи)
│   └── server/             # Fastify (порт 8787 dev)
│       ├── drizzle/        # миграции
│       └── src/
│           ├── index.ts    routes/  ws/  db/  services/  plugins/
│           └── lib/        # scoring-adapter (из shared), hash-lock
├── packages/
│   ├── shared/        # Zod-контракты + чистые функции: scoring, economy, levels (см. SPEC 02/04)
│   ├── scenario-gen/  # детерминированный генератор миссий (seedrandom)
│   └── content/       # scenarios/*.json, cards.json, entities.json, protocols.json,
│                      # academy/*, locales/*(-> client), datasets/registry.json, токены
└── tests/  # playwright.config.ts, e2e-флоу, fixtures server
```

Импорты: `apps/*` могут импортировать `packages/*`; `packages/*` не импортируют `apps/*`; `shared` чистый (без phaser, без node API в публичной поверхности — выносить в subpath `@arena/shared/node` при необходимости).

## 2. Клиент: инициализация и game loop

- `main.ts`: feature-detect WebGL → `new Phaser.Game(cfg)`; при отсутствии WebGL — HTML-fallback экран «устройство не поддерживается» (делается до прелоудера).
- `config`: `type: Phaser.WEBGL`, `width: 420, height: 900` (design space, mobile portrait), `scale: { mode: FIT, autoCenter: CENTER_BOTH }`, `fps smooth`, `backgroundColor: '#05070F'`.
- Рендер-цикл: сцены рисуют UI через rexUI/Graphics; **перерисовка по dirty-флагам**, не на каждый кадр (бюджет — 07_TESTING §6). Виртуальные градиенты/глoу — заранее запечённые текстуры, не runtime-шейдеры (кроме отдельно согласованных).
- Сцены запускаются стеком: одна «page-сцена» + оверлеи (`Modal`, `Toast`) поверх. Переходы — через `SceneRouter` (единое место, fade/slide пресеты).

## 3. Сцены и их ответственность

| Сцена | Ключ | Назначение |
|-------|------|-----------|
| BootScene | `boot` | минимальный прелоуд лого/шрифтов, регистр плагинов |
| PreloadScene | `preload` | атласы, звуки, контент-пак, прогресс-бар |
| HubScene | `hub` | вход в режимы, дейли-виджет, профиль-сводка |
| MissionScene | `mission` | ядро: TopBar, ситуация, BrowserWidget, CardTray, DecisionBar, ExtraStepDock |
| RevealScene | `reveal` | play-forward истории, маркер решения, ожидание скоринга |
| DebriefScene | `debrief` | 3 слоя результата + разбор + награды + CTA |
| AcademyScene | `academy` | главы/уроки |
| BestiaryScene | `bestiary` | список/деталь сущностей |
| JournalScene | `journal` | паттерны, история решений |
| MarketScene | `market` | витрины (P8) |
| TournamentScene | `tournament` | турниры (P7) |
| SettingsScene | `settings` | профиль/звук/локаль |
| Overlay-сцены | `modal`, `toast` | не page-сцены; sleep/wake |

## 4. Состояние (Zustand vanilla, без React)

Сторы (все — `createStore` из `zustand/vanilla`, persist → `localStorage` через `createJSONStorage`, версии миграций обязательны):

| Стор | Ответственность | Persist |
|------|-----------------|---------|
| `sessionStore` | токен гостя, режим онлайн/офлайн, WS-статус | частично |
| `playerStore` | профиль, XP, уровень, streak, настройки | да |
| `contentStore` | загруженный контент-пак, версии, выбор миссии | метаданные |
| `missionStore` | эпhemeral: текущая миссия, выбранные карты, открытые вкладки, таймер, extra-step | нет (восстановление сессии P6+) |
| `economyStore` | монеты, $SIG-кредиты, инвентарь косметики | да |
| `netStore` | очередь офлайн-результатов для sync | да |

Правила: (1) сторы **не импортируют Phaser**; (2) сцены читают сторы и подписываются через `bridge/EventBus` (`store → emit('player:xp', v)`); (3) запись в стор — только из actions стора или netClient, не из UI-обработчиков напрямую; (4) вся игровая математика живёт в `shared` (чистые функции), стор — только держатель.

## 5. CandleChart (кастомный, на Graphics) — `apps/client/src/game/chart/`

Вход: `Candle[]` (SPEC 02) + конфиг `{ volume?: VolumeBar[], levels?: LevelLine[], decisionIndex?, mode: 'historic'|'reveal' }`.

- **Draw pipeline** (порядок): фон-сетка → level-линии (штрих) → свечи → объёмы (нижняя полоса 18–22% высоты) → маркер decision point (кольцо+пульс) → fog-слой (в `historic` закрывает всё после `decisionIndex`: тёмная плашка с «scanline» полосами и надписью-заглушкой из тона игры) → ось цены справа (pooled Text) → время снизу (редкие метки).
- Свечи: `Graphics#generateTexture` по (up/down × body/wick) → спрайты из пула; объёмы — один Graphics на redraw. Бюджет: полный redraw ≤ 2 мс при 120 свечах.
- **Reveal-режим**: `revealTo(index)` — твином двигается фронт, свечи «дорисовываются» по одной (тик-звук на каждую), fog отползает. Скорость настраивается, кнопка «пропустить» обязана быть.
- API: `setData()`, `setDecisionMarker(i)`, `setMode()`, `revealTo(i, {speed})`, `destroy()`; никаких DOM-элементов, никакого lightweight-charts.
- Детерминизм рендера: одинаковые данные → пиксельно одинаковый кадр (основа visual-тестов Playwright).

## 6. rexUI и UI-слой

- Плагин подключается в BootScene: `rexUI = scene.plugins.get('rexUI')` при `use`-решении ADR-001; при `fallback` — внутренний `ui-kit` с той же поверхностью (Panel/TabPages/Slider/ScrollablePanel/Badge/Toast), реализованной на контейнерах+Graphics.
- Тема — `rexui-theme.ts`: все цвета/радиусы из `design-tokens.json` (SPEC 06). Запрещены литеральные hex вне токенов (lint-правило-проверка в code review, автоматизация приветствуется).

## 7. Сервер (Fastify)

- `plugins/`: `env` (Zod-валидация .env), `db` (drizzle better-sqlite3), `auth` (guest token: `POST /api/auth/guest` → `{playerId, token}`; JWT или подписанный opaque), `ratelimit`.
- REST: `POST /api/auth/guest` · `GET /api/missions/next?mode=arena` · `GET /api/missions/:id/package` (публичная часть, БЕЗ fore) · `POST /api/attempts` (decision+extra+cards+ts) → `{ attemptId, score, revealToken }` · `GET /api/attempts/:id/reveal?token=` (fore + ground truth) · `GET /api/profile` · `GET /api/journal` · турнирные (P7) · маркет (P8).
- WS `/ws`: каналы `profile:update`, `economy:update`, `tournament:*`; миссия тоже может идти по WS (тот же payload, другой транспорт). Сообщения — Zod-конверты SPEC 02 §7.
- `services/scoring`: только серверная валидация итогового скора: клиентский preview считается той же функцией из `@arena/shared`, итоговая запись — сервером (`attempts.score`).
- БД таблицы (Drizzle): `players`, `content_versions`, `missions` (кэш отдачи), `attempts`, `player_patterns`, `inventory`, `purchases_stub`(P8), `tournaments`, `tournament_entries` (P7), `datasets` (версия+hash+source).
- Логирование pino (level из env), request-id в заголовках.

## 8. Детерминизм и scenario-gen

- Единственный источник случайности — `seedrandom(seed)`, где `seed` присылает сервер (или офлайн-резолвер из контент-пака). **Запрещены** `Math.random()`, `Date.now()` внутри генерации/скоринга (lint: `no-restricted-globals` для math-random в packages/scenario-gen и shared; тест «один seed → один JSON» с snapshot).
- `scenario-gen` получает `datasetSlice + contentDef + seed` → выдаёт конкретный сценарий (смещения, шумовые формулировки, подмешивание дистракторов). Тот же вход → тот же выход, побайтово.
- Версия генератора входит в `contentVersion`; смена версии = миграция/перегенерация офлайн-паков.

## 9. Честность, скрытие будущего, античит

1. Датасет сценария делится: `past` (до t0) / `fore` (после). В выдаче клиенту — только `past` + `foreHash` (`sha256(JSON(fore)+salt)`), `fore` остаётся на сервере.
2. `POST /api/attempts` фиксирует решение (server ts) → только потом `/reveal` отдаёт `fore` + исходный `foreHash` и `salt` (клиент может проверить хэш — «проверяемая честность»).
3. Обфускация до решения: `assetAlias` (напр. «COIN-PERP»), сдвиг ценовой шкалы (масштаб ±k, круглые «ценовые» уровни заменены), размытие дат («день 214»), вырезание уникальных заголовков. После reveal — показываются реальные, если допустимо лицензией датасета.
4. Детерминизм запрещает «подгонку рынка»: исход хранится до ответа. Тест `anticheat.spec` обязан ломаться, если fore попадает в payload до attempts.
5. Турниры (P7): набор заданий фиксируется сервером до старта, reveal откладывается до закрытия окна (или до попытки участника), скорость засчитывается только tie-breaker при равном качестве.

## 10. Офлайн-режим

- Контент-пак (N сценариев bundle'ится в PWA) + офлайн-резолвер: scenario-gen + scoring исполняются локально (та же `@arena/shared`), результаты копятся в `netStore.outbox` и синкаются при сети (идемпотентность по `attemptId`).
- Офлайн-режим — усечённый: без турниров/маркета; честность доверительная (device-side), о чём прямо говорит UX («офлайн-результаты не участвуют в рейтингах»).

## 11. PWA

- vite-plugin-pwa (`generateSW` на старте; перейти на `injectManifest` при потребности в роутинге кэша): precache app shell + контент-пак версии, runtime-cache для API GET (stale-while-revalidate), страница обновления «новая версия — обновить?».
- Manifest: имя Signal Arena, темы из токенов, иконки 3 уровней (из бренд-системы), display standalone.

## 12. Среды и порты

Dev: client 5173 (`server.host 0.0.0.0`, allowedHosts включает песочные превью-домены `*.e2b.app`), server 8787, proxy `/api`,`/ws` через Vite → 8787 (браузер никогда не ходит на localhost напрямую — только относительные URL через прокси). Preview/prod: статика клиента + server на одном origin.

## 13. CI (T-P0-06)

GitHub Actions: install (pnpm, cache) → typecheck → lint → unit (vitest, coverage threshold по 07 §5) → build (client+server) → Playwright e2e (chromium, webServer поднимает оба) → артефакты: отчёты, скриншот-диффы. Прайсинг времени: цель прогона ≤ 6 мин.
