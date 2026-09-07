# tradingcrypto_fromzerotohero → **SIGNAL ARENA**

Мобильная PWA-игра: тренажёр **качества торговых решений** на исторических ситуациях крипторынка. Не терминал, не демо-счёт, не курс с тестами. Игрок анализирует «браузер трейдера», принимает решение из 4 вариантов — и только потом видит заранее зафиксированное продолжение истории. Награда — за качество мышления (0–100), а не за угаданную свечу.

## 🤖 Работа с ИИ-агентами (главное)

Проект разрабатывается **эстафетой ИИ-агентов**. Если ты агент — **начни с [`AGENTS.md`](./AGENTS.md)**, там полный протокол. Пояснения от человека не требуются.

| Что сейчас | Где смотреть |
|---|---|
| Живой статус проекта (фаза, %, кто что делает) | [`docs/status/STATUS.md`](./docs/status/STATUS.md) |
| Хроника всех сессий (append-only) | [`docs/status/HANDOFF_LOG.md`](./docs/status/HANDOFF_LOG.md) |
| Доска задач 0→100% (~118 задач с DoD и тестами) | [`docs/roadmap/TASK_BOARD.md`](./docs/roadmap/TASK_BOARD.md) |
| Дорожная карта фаз P0→P10 | [`docs/roadmap/ROADMAP.md`](./docs/roadmap/ROADMAP.md) |
| Техническое задание (8 спецификаций) | [`docs/spec/`](./docs/spec/) |
| Готовые промпты для запуска агентов | [`docs/prompts/BOOTSTRAP_AGENT.md`](./docs/prompts/BOOTSTRAP_AGENT.md) |

## Стек

Phaser 4 + TypeScript + Vite (WebGL) · rexUI · кастомный CandleChart на Graphics · Zustand (vanilla) · seedrandom + scenario-gen · Fastify 5 + SQLite + Drizzle + Zod + WS · Phaser Sound · vite-plugin-pwa · Vitest + Playwright · pnpm workspaces.

## Источники истины (master-ветка)

Продуктовая конституция, закон текстов (панк-таблоидная криптосатира), айдентика (S + инь-янь бык/медведь) и 60+ UI/арт-референсов лежат в ветке `master` — см. перечень в [`docs/spec/00_OVERVIEW.md`](./docs/spec/00_OVERVIEW.md) §9.

## Быстрый старт (появится после фазы P0)

```bash
pnpm i
pnpm dev        # client http://localhost:5173, server :8787
pnpm -w test && pnpm -w typecheck && pnpm e2e
```
