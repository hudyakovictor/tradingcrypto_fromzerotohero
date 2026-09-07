# SPEC 07 — TESTING STRATEGY

Для эстафетной разработки тест = единственный объективный пропуск в DONE. Философия: **контракты и математика — 100% покрываем; сцены — e2e по пользовательским флоу; визуал — снапшот Phaser-кадров с допуском.**

## 1. Пирамида

| Уровень | Инструмент | Что | Порог |
|---|---|---|---|
| Unit | Vitest | shared: scoring, rules-DSLite, economy, levels, selector; scenario-gen | lines ≥ 85% packages/*; **scoring/генератор = 100% веток** |
| Contract | Vitest + Zod | все fixtures парсятся; round-trip; антилик-обход Public | 100% схем, фикстуры обязательны к каждой фиче контракта |
| Determinism | Vitest | один `(seed, version)` → побайтово тот же сценарий/скор (snapshot) | обязателен к любому изменению генератора |
| Integration (server) | Vitest + fastify.inject | REST маршруты, WS-конверты, hash-lock честности, drizzle-миграции up/down | каждый эндпоинт |
| E2E | Playwright (chromium) | флоу §3 | зелёные на CI к каждой клиентской задаче |
| Visual | Playwright screenshots | ключевые сцены в фиксированном состоянии (420×900) ≤ 0.5% pixel diff | на сценах: mission, debrief, hub, academy, bestiary |

## 2. Обязательные гейты (команды, root scripts)

```bash
pnpm -w typecheck     # tsc --noEmit по всем пакетам (strict)
pnpm -w lint          # eslint, включая no-math-random в scenario-gen/shared
pnpm -w test          # vitest run, thresholds падают сборку
pnpm -w test:e2e      # playwright (поднимает client+server на тест-портах)
pnpm build            # клиент (PWA) + сервер
```

DONE = все применимые строчки зелёные. Не применима (например, e2e для чистого контент-PR) — пишем в лог причину.

## 3. E2E флоу (минимальный набор; растёт с фазами)

1. `boot.spec` — WebGL-экран, прелоадер доходит до Hub.
2. `onboarding.spec` — 10 шагов, пропуск, флаг завершения в профиле.
3. `mission.spec` — открыть миссию → открыть 2 вкладки → выбрать карту → решение → (extra) → commit → Reveal → Debrief показывает 3 слоя и score 0–100.
4. `honesty.spec` — перехват сети: в payload миссии НЕТ fore (глубокий обход), после attempts reveal отдаёт fore + валидный foreHash (пересчёт на клиенте совпадает).
5. `offline.spec` — отключить сеть после загрузки: миссия из контент-пака проходится, результат ложится в outbox, после сети — sync.
6. `journal.spec` — после 2 миссий записи в журнале, паттерн-счётчики изменились.
7. `academy.spec` — глава открывается, урок читается, связка к сценарию ведёт в Арену.
8. `paywall.spec` (P8) — ни один платный SKU не влияет на scoring/data (проверка флагами + контрактный тест).
9. `tournament.spec` (P7) — два аккаунта, одинаковый seed-набор, tie-break по правилам.
10. `pwa.spec` — manifest валиден, SW ставится, офлайн-старт повторного визита.

## 4. Данные для тестов

- `packages/shared/fixtures/` каноничны (SPEC 02 §9). Playwright использует **отдельный тестовый контент-пак** с короткими сценариями (fore 12 свечей) и детерминированными seed.
- Сервер в e2e — `NODE_ENV=test`, БД temp-файл, seed через `pnpm db:seed:test`.
- Запрещено мокать `shared` в unit-тестах самого shared.

## 5. Покрытие и мутационные проверки

- Vitest coverage thresholds: `shared` lines 95 / branches 90 (scoring 100/100); `scenario-gen` 90/85; `server/services` 85; client `stores` 80.
- Раз в фазу (на гейте) — мутационный спот-чек scoring (opционально Stryker, не блокирует).
- Любой багфикс = сначала падающий тест, потом фикс (правило репозитория).

## 6. Производительность как тест (P4+)

- Playwright-пробник: средний кадр MissionScene на эмулируемом mid-tier (CPU throttle 4×) > 50 fps; полный redraw чарта ≤ 2 мс (лог из `performance.now` в хуке `window.__arena.perf`).
- Бюджет бандла: initial JS ≤ 1.6 МБ brotli, контент-пак MVP ≤ 8 МБ.

## 7. Матрица устройств e2e

iPhone 13 (390×844), Pixel 7 (412×915), desktop fallback 420×900 framed. Locale-прогоны: RU обязательно, EN на smoke.

## 8. Что НЕ тестируем

- Точный внешний вид сгенерированного арта (только наличие/загрузку).
- Сторонние CDN (их нет в рантайме по умолчанию).
- «Качество обучения» — это полевой плейтест (чек-лист в P4), не автотест.
