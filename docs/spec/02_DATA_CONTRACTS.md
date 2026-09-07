# SPEC 02 — DATA CONTRACTS (истина по структурам данных)

Все контракты — **Zod-схемы в `packages/shared/src/contracts/`**, из них генерируются TS-типы (`z.infer`). Клиент и сервер импортируют только отсюда. Изменение схемы = ADR + bump `CONTRACT_VERSION`. Любой JSON, проходящий границу (сеть/диск), валидируется `.parse()`.

## 0. Соглашения

- Временные метки: `epochMs` (number). Все «длительности» в ms.
- Деньги/цены: number в float допустим только на рендере; в контрактах — `number`, округление до `priceDecimals` датасета.
- ID: `string`, формат `^[a-z0-9][a-z0-9\-_]{2,48}$`.
- Локализуемые строки: ключ i18n (`i18nKey`) ИЛИ `LocalizedText = { ru: string; en?: string }` для авторского контента. UI-тексты — только через i18nKey.
- `CONTRACT_VERSION: "0.1.0"` на старте; semver: breaking → minor/major, добавление опциональных полей → patch.

## 1. Рыночные данные

```ts
Candle = { t: epochMs; o: number; h: number; l: number; c: number; v: number }
VolumeBar = { t: epochMs; v: number; dir: 'up' | 'down' }
LevelLine = { price: number; kind: 'support' | 'resistance' | 'liquidationZone' | 'custom'; labelKey?: string }
MarketSlice = {
  symbolReal: string            // напр. "BTC/USDT PERP" — НЕ отдаётся клиенту до reveal
  symbolShown: string           // обфускация, напр. "ALPHA-PERP"
  timeframe: '1m'|'5m'|'15m'|'1h'|'4h'|'1d'
  priceDecimals: number
  priceScale: number            // множитель обфускации отображаемых цен (1 = без шкалы)
  past: Candle[]                // всё до t0 включительно — ОТДАЁТСЯ клиенту
  fore: Candle[]                // после t0 — ТОЛЬКО на сервере до фиксации решения
}
DatasetRef = { datasetId: string; version: string; source: string; sha256: string }
```

## 2. Сценарий (ядро продукта)

```ts
TabKind = 'chart' | 'higherTf' | 'volumeLiquidity' | 'news' | 'sentiment'
        | 'macro' | 'tokenData' | 'onchain' | 'position' | 'journalSelf'

SourceTab = {
  id: string; kind: TabKind; iconKey: string
  titleKey: string                      // i18n
  payload: TabPayload                   // дискриминированный union по kind:
}                                       // chart → { slice: PublicSlice }
                                        // news → { items: NewsItem[] } (заголовки обфусцированы)
                                        // sentiment → { fgi?: number; social: {...} }
                                        // onchain → { flows: FlowItem[]; unlocks?: ... }
                                        // position → { openPosition?: PositionState } (тип "внутри позиции")
                                        // volumeLiquidity → { levels, wallHint?, funding?, oi? }
                                        // … per-kind схемы фиксируются в contracts/tabs.ts

SkillCardType = 'read' | 'decide' | 'protect'
SkillCardRef = { cardId: string; recommended?: boolean }  // recommended допускается только на низких уровнях

DecisionOption = {
  key: 'A' | 'B' | 'C' | 'D'
  labelKey: string
  // ниже — скрытые от клиента до reveal поля хранятся отдельно (rubric-side)
}

ExtraStep =
  | { kind: 'riskSize';    minPct: number; maxPct: number; stepPct: number }
  | { kind: 'invalidation'; options: { id: string; labelKey: string }[] }
  | { kind: 'confirmation'; options: { id: string; labelKey: string }[] }
  | { kind: 'evidencePick'; maxPicks: number }   // отметить релевантные доказательства
  | { kind: 'pauseChoice'; options: { id: string; labelKey: string }[] }

ProtocolDef = { id: string; nameKey: string; ruleKey: string; scoringHook: string } // см. LOGIC §5

ScenarioPublic = {                      // то, что УХОДИТ клиенту до решения
  scenarioId: string; contentVersion: string; contractVersion: string
  briefKey: string                      // короткий контекст (i18n)
  situationType: 'preEntry' | 'inPosition'
  difficultyTier: 1|2|3|4|5             // внутренняя градация для матчмейкинга контента
  tabs: SourceTab[]                     // 1..7 — только влияющие на решение (SPEC 00 §10)
  cards: SkillCardRef[]
  decisions: [DecisionOption, DecisionOption, DecisionOption, DecisionOption]
  extraStep?: ExtraStep
  protocolId?: string
  entityIdsHiddenTier: 'named' | 'shown' | 'hiddenAfterReveal'  // конституция §7
  market: PublicSlice                   // fore заменён заглушкой { count, foreHash, saltHash }
  timerMs?: number                      // редко; скрытый/явный — никогда не основной критерий
  foreHash: string                      // sha256(canonical(fore)+salt)
}

ScenarioSecret = {                      // живёт ТОЛЬКО на сервере / в контент-бандле до reveal
  scenarioId: string; salt: string
  fore: Candle[]; foreEvents?: { atIndex: number; labelKey: string }[]
  scoringRubric: ScoringRubric          // §3
  acceptableDecisions: { keyedBy: DecKey; quality: 'optimal'|'acceptable'|'poor'|'harmful' }[]
  debriefContent: { factKey: string; logicKey: string; ruleKey: string; theoryRef?: string }
  entityIds: string[]; entityRevealedAfter: boolean
  revealMeta: { realSymbol: string; realDateNoteKey?: string; sources: string[]; methodNoteKey?: string }
}
```

## 3. Скоринг-рубрика (значения — в LOGIC §2)

```ts
ScoringRubric = {
  weights: { context: number; evidence: number; action: number; risk: number; discipline: number } // Σ = 100
  actionQuality: Record<DecKey, 0|25|50|75|100>        // базовые очки за выбор
  riskRules: { condition: string; delta: number }[]     // детерминированные правила DSL-lite
  disciplineRules: { condition: string; delta: number }[]
  evidenceRules: { tabId: string; delta: number }[]     // засчитывается факт открытия вкладки? — НЕТ:
}                                                       // evidence оценивается по связи решения с payload'ом,
                                                        // открытие вкладки само по себе очков не даёт (антиметроном)
ScoreBreakdown = {
  total: number                       // 0..100
  parts: { context: number; evidence: number; action: number; risk: number; discipline: number }
  verdict: 'optimal'|'acceptable'|'poor'|'harmful'
  notes: string[]                     // i18n-ключи объяснений
}
```

## 4. Попытка и раскрытие

```ts
AttemptSubmit = { scenarioId: string; clientSeed: string; decisionKey: DecKey;
  cardIdsUsed: string[]; extra?: ExtraStepAnswer; durationMs: number; contentVersion: string }
AttemptResult = { attemptId: string; score: ScoreBreakdown; revealToken: string;
  xpDelta: number; creditsDelta: number; streakState: StreakState; entityDefeated?: string[] }
RevealPackage = { fore: Candle[]; foreEvents?: …; secret: ScenarioSecret (минус внутр. поля);
  foreHash: string; salt: string }     // клиент САМ перепроверяет sha256 и сверяет с foreHash из Public
```

## 5. Контентные справочники (`packages/content`)

```ts
SkillCardDef = { id: string; type: SkillCardType; nameKey: string; flavorKey: string;
  iconKey: string;                    // иконка-объект средней детализации (SPEC 06 §5)
  tier: 1|2|3; apCost?: 0|1|2|3;      // apCost существует ТОЛЬКО если механика AP утверждена (LOGIC §7)
  unlock: { level?: number; chapterId?: string }; theoryRef?: string }

EntityCategory = 'market' | 'emotion' | 'infonoise' | 'risk' | 'web3'
EntityDef = { id: string; category: EntityCategory; nameKey: string; flavor: { nameKey, quoteKey };
  artKey: string; threatBand: [number, number];      // появление по диапазону уровней
  manifestationKey: string; weaknessKey: string; counterCardIds: string[]; relatedErrorKeys: string[] }

ProtocolDef — §2 выше. Chapter = { id; titleKey; lessons: Lesson[]; unlocksCardIds; unlocksScenarioIds }
Lesson = { id; titleKey; bodyKey; estSec: number (≤90); relatedCardIds; relatedScenarioIds }
```

Каждый справочник — JSON с `$schema`-пометкой, валидируется в CI (T-P5-02). Стартовые пулы — LOGIC §8/CONTENT §6.

## 6. Профиль, экономика, журнал

```ts
PlayerProfile = { playerId: string; handle: string; xp: number; level: number; // 0..99
  streak: { days: number; lastClosedErrorDate?: string };   // streak = ДЕНЬ С ЗАКРЫТОЙ ОШИБКОЙ
  patternCounts: Record<string, number>;     // errorKey -> count (профиль повторяющихся ошибок)
  skillMastery: Record<string, 0|1|2|3>;     // cardId -> звёзды
  entitiesDefeated: string[]; createdAt: epochMs }

EconomyState = { coins: number; sigCredits: number; ownedCosmetics: string[];
  // нет и никогда: cash, realMoneyBalance, withdrawable
}

JournalEntry = { attemptId; scenarioId; ts; score: ScoreBreakdown; decisionKey;
  errorKeys: string[]; ruleLearnedKey?: string }
PatternInsight = { errorKey: string; count: number; lastSeen: epochMs;
  recommendedScenarioIds: string[]; closedFlag: boolean }  // «закрытая ошибка» = N подряд без повтора (LOGIC §6)
```

## 7. WS-протокол (конверт)

```ts
WsEnvelope<T> = { v: 1; type: string; ts: epochMs; payload: T }
// client → server:  'mission:next' {mode} | 'attempt:submit' AttemptSubmit | 'ping'
// server → client:  'mission:package' ScenarioPublic | 'attempt:result' AttemptResult
//                   | 'economy:update' EconomyState | 'tournament:update' (P7) | 'error' {code, messageKey}
```

REST и WS используют ОДНИ payload-схемы. Коды ошибок: `ERR_RATE`, `ERR_CONTENT_VERSION`, `ERR_ATTEMPT_DUP`, `ERR_REVEAL_LOCKED`, `ERR_VALIDATION`, `ERR_UNAVAILABLE_OFFLINE`.

## 8. Полный пример сценария (сокращённый, эталон для фикстур)

```jsonc
{
  "scenarioId": "scn_breakout_no_volume_001",
  "contentVersion": "0.4.0", "contractVersion": "0.1.0",
  "briefKey": "scenario.bnv001.brief",                 // «Пробой без объёма. Объём молчит.»
  "situationType": "preEntry", "difficultyTier": 1,
  "tabs": [
    {"id":"t_chart","kind":"chart","iconKey":"tab.candles","titleKey":"tab.chart",
     "payload":{"slice":{"symbolShown":"ALPHA-PERP","timeframe":"15m", "…":"past+candles+volume+resistance level"}}},
    {"id":"t_vol","kind":"volumeLiquidity","iconKey":"tab.bars","titleKey":"tab.volume",
     "payload":{"levels":[],"funding":-0.0001,"oiDeltaPct":-1.8}}
  ],
  "cards": [ {"cardId":"card_trend_check"}, {"cardId":"card_volume_confirm","recommended":true},
             {"cardId":"card_stoploss_discipline"}, {"cardId":"card_wait_retest"} ],
  "decisions": [
    {"key":"A","labelKey":"dec.enterLongNow"},
    {"key":"B","labelKey":"dec.waitRetestVolume"},
    {"key":"C","labelKey":"dec.shortFakeout"},
    {"key":"D","labelKey":"dec.stayOut"}
  ],
  "extraStep": {"kind":"riskSize","minPct":0.5,"maxPct":5,"stepPct":0.5},
  "entityIdsHiddenTier": "shown",
  "market": { "…": "public slice", "count": 40, "foreHash": "…", "saltHash": "…" },
  "secret": null  // в Public-выдаче отсутствует физически, поле не сериализуется
}
```

Секретная часть эталонного сценария: fore = импульс +12% после ретеста (B — optimal 100; A — poor: догонялка без подтверждения; D — acceptable; C — harmful без стоп-условия), сущность `ent_fake_breakout_phantom`, debrief: факт → логика → правило «пробой без объёма — приглашение. Приглашают не тебя.»

## 9. Фикстуры и тесты контрактов

- `packages/shared/fixtures/` — ≥ 3 полных сценария (preEntry, inPosition, extraStep=evidencePick), сущности, 12 карт.
- Тест `contracts.spec.ts`: каждый фикстурный JSON проходит `.parse()`; round-trip serialize→parse стабилен; Public-JSON не содержит ключей `fore`, `scoringRubric`, `acceptableDecisions` (глубокий обход — антилик).
