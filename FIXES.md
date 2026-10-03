# Исправления для Issues #12 и #13

## Issue #13: Бот не активирует сделки (Arbitrage и DipArb)

### Проблема
Стратегии Arbitrage и DipArb были отключены по умолчанию. Они требовали явной установки переменных окружения:
- `ARBITRAGE_ENABLED=true`
- `DIPARB_ENABLED=true`

### Исправление
Изменены значения по умолчанию в `bot-with-dashboard.ts:111-127`:
```typescript
arbitrage: {
  enabled: process.env.ARBITRAGE_ENABLED !== 'false',  // Теперь включено по умолчанию
  // ...
},

dipArb: {
  enabled: process.env.DIPARB_ENABLED !== 'false',  // Теперь включено по умолчанию
  // ...
},
```

Теперь стратегии **включены по умолчанию** и могут быть отключены через:
- `ARBITRAGE_ENABLED=false`
- `DIPARB_ENABLED=false`

---

## Issue #12: Dry-run режим не отслеживает позиции и историю

### Проблема
В dry-run режиме:
1. ✗ Счетчики обновляются, но панель Positions остается пустой
2. ✗ Session History не содержит записей об открытии позиций
3. ✗ Нет симуляции закрытия позиций при резолюции рынка
4. ✗ Arbitrage: только мгновенная оценка PnL, без открытия позиций
5. ✗ DipArb: сигналы есть, но позиции не отслеживаются
6. ✗ Smart Money: только счетчики, без детальных записей

### Исправления

#### 1. Добавлен тип `PaperPosition` (`src/dashboard/types.ts:97-117`)
```typescript
export interface PaperPosition {
  id: string;
  timestamp: string;
  strategy: 'smartMoney' | 'arbitrage' | 'dipArb' | 'direct';
  market: string;
  tokenId: string;
  side: 'YES' | 'NO' | 'UP' | 'DOWN';
  size: number;
  entryPrice: number;
  costBasis: number;
  status: 'open' | 'closed';
  closePrice?: number;
  closeTimestamp?: string;
  realizedPnL?: number;
  closeReason?: 'manual' | 'resolution' | 'strategy';
}
```

#### 2. Обновлен `BotState.paper` (`src/dashboard/types.ts:88-95`)
Добавлено поле `positions: PaperPosition[]` для отслеживания открытых и закрытых позиций.

#### 3. Добавлены функции управления позициями (`bot-with-dashboard.ts:489-603`)
- `openPaperPosition()` - открывает бумажную позицию
- `closePaperPosition()` - закрывает позицию с расчетом PnL
- `settlePaperPositionsOnResolution()` - закрывает позиции при резолюции рынка

#### 4. Обновлены стратегии:

**Arbitrage** (`bot-with-dashboard.ts:975-997`):
- ✓ Открывает обе позиции (YES и NO) при обнаружении возможности
- ✓ Записывает позиции в историю
- ✓ Правильно рассчитывает PnL

**DipArb** (`bot-with-dashboard.ts:1119-1211`):
- ✓ Открывает позицию на Leg1 (вход)
- ✓ Открывает хедж на Leg2
- ✓ Закрывает позиции при завершении раунда
- ✓ Обрабатывает таймаут-выходы

**Smart Money** (`bot-with-dashboard.ts:748-768`):
- ✓ Открывает позиции при копировании BUY ордеров
- ✓ Закрывает позиции при получении сигнала о закрытии
- ✓ Рассчитывает реальный PnL

#### 5. Обновлена инициализация (`bot-with-dashboard.ts:1671-1677, 1728-1737`)
При включении dry-run режима `state.paper.positions` инициализируется пустым массивом.

---

## Результат

### Issue #13 ✓ Исправлено
Бот теперь активирует сделки для Arbitrage и DipArb по умолчанию.

### Issue #12 ✓ Исправлено
Dry-run режим теперь:
- ✓ Отслеживает все открытые позиции в `state.paper.positions`
- ✓ Записывает открытие и закрытие позиций в Session History
- ✓ Правильно рассчитывает PnL для каждой позиции
- ✓ Поддерживает все стратегии (Arbitrage, DipArb, Smart Money, Direct)
- ✓ Позволяет просматривать детальную историю сделок

---

## Тестирование

Для проверки исправлений:

1. Убедитесь, что установлены зависимости:
```bash
npm install
cd dashboard && npm install && npm run build && cd ..
```

2. Создайте `.env` файл с вашими настройками:
```env
POLYMARKET_PRIVATE_KEY=0xYourPrivateKeyHere
CAPITAL_USD=250
DRY_RUN=true
```

3. Запустите бота:
```bash
npx tsx bot-with-dashboard.ts
```

4. Откройте дашборд в браузере (URL будет показан в терминале)

5. Проверьте:
   - ✓ Стратегии Arbitrage и DipArb включены (зеленые индикаторы)
   - ✓ Позиции появляются на вкладке Positions
   - ✓ История сделок записывается на вкладке Session History
   - ✓ PnL правильно обновляется

---

## Изменённые файлы

1. `src/dashboard/types.ts` - добавлен тип `PaperPosition`
2. `bot-with-dashboard.ts` - основные исправления:
   - Строки 111-127: значения по умолчанию для стратегий
   - Строки 489-603: функции управления позициями
   - Строки 975-997: обновлен Arbitrage
   - Строки 1119-1211: обновлен DipArb
   - Строки 748-768: обновлен Smart Money
   - Строки 1671-1677, 1728-1737: инициализация

---

**Дата исправления**: 2026-10-03
**Автор**: Claude Opus 4.8
