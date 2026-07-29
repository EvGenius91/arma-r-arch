# EntityLockRegistry

Назад: [[Architecture.md]]

Отдельный сервис на игровом сервере: **реестр локальных блокировок сущностей**. Не часть EntityManager и не часть TradeManager — самостоятельный компонент, от которого они зависят.

## Зависимости

```text
EntityManager  ──depends on──►  EntityLockRegistry
TradeManager   ──depends on──►  EntityLockRegistry
```

- [[EntityManager.md|EntityManager]] ставит uid в реестр через `Lock` **когда пачка ops по этому uid уходит на бекенд (in-flight)** и снимает через `Unlock` после ответа. Пока ops только в исходящей очереди (ещё не ушли) — EM **не** держит lock; см. [[EntityManager.Operations.md]].
- [[TradeManager.md|TradeManager]] на время продажи ставит `SellPending` в тот же реестр, чтобы EntityManager не принял дроп/передачу продаваемого uid.
- Реестр **не** знает о содержимом контейнеров и не ходит на бекенд — только занятость `entityUid`.

## Назначение

Пока сущность занята (пачка EntityManager **in-flight**, либо идёт продажа через Trade), повторный dispose / pick / sell локально запрещён:

- EntityManager не даст выбросить / переложить / подобрать / передать uid, пока по нему летит пачка;
- Trade не начнёт `sell`, если uid уже в lock (in-flight EM или чужой `SellPending`);
- пока ops лишь в очереди EM (lock ещё нет) — второй игрок **может** подобрать предмет после локального оптимизма дропа; обе ops упорядоченно уйдут в одной пачке.

Реестр **не** источник истины бекенда и **не** отменяет CommandBus: `removeEntity` с бекенда выполняется даже для заблокированного uid; после удаления запись из реестра снимается.

---

## Связь с очередью EntityManager

```text
EntityManager.enqueuePickupEntity / enqueueDropEntity / enqueueMoveEntity
  → EntityLockRegistry.IsLocked(entityUid)?  // да (in-flight / SellPending) → отказ
  → локальный оптимизм + запись в очередь EntityManager
  → … таймер ~1 с или barrier …
  → EntityLockRegistry.Lock(entityUid, reason, scope, ownerService: EntityManager)
  → отправка пачки на бекенд (in-flight)
  → Ok / Fail
  → EntityLockRegistry.Unlock(entityUid)
```

`Lock` от EntityManager — на окно in-flight, не на всё время нахождения ops в очереди. Подробности flush / barrier / порядка пачки: [[EntityManager.Operations.md]]. Сигнатуры enqueue — в [[EntityManager.md#enqueuedropentity|EntityManager.md]].

---

## Кто вызывает

| Сервис | Когда |
|--------|--------|
| [[EntityManager.md\|EntityManager]] | `IsLocked` на `enqueue*` (отказ, если уже in-flight / SellPending); `Lock` при уходе пачки в in-flight; `Unlock` после Ok/Fail; очистка при hard-reset / `removeEntity` |
| [[TradeManager.md\|TradeManager]] | перед / в момент старта продажи — `IsLocked`; если свободен — `Lock(…, SellPending)`; после Fail sell — `Unlock`; после Ok sell — `Unlock` когда мир догнал / вместе с обработкой `removeEntity` |
| Мир / UI (опционально) | только `IsLocked` для подсказок; не пишут в реестр в обход менеджеров |

EntityManager **не владеет** реестром как внутренней приватной структурой — он **зависит** от сервиса и вызывает его API.

---

## Структура записи

```text
lockedEntities[entityUid] = {
  reason: PickUp | Drop | Move | Transfer | SellPending | ParentOfPendingChild | …,
  scope: InventoryOps | ContainerDispose,
  ownerService: EntityManager | TradeManager | …,
  characterUid: string,
  pendingOperationId: string | null,
  since: timestamp
}
```

| Поле | Смысл |
|------|--------|
| `reason` | Почему занято (дроп, продажа, …) |
| `scope` | Какой класс действий запрещён |
| `ownerService` | Кто поставил lock (для отладки и правил Unlock) |
| `characterUid` | Кто инициировал |
| `pendingOperationId` | Связь с очередью EntityManager, если есть |
| `since` | Время постановки (таймауты / диагностика) |

---

## Методы

```text
Lock(entityUid, reason, scope, ownerService, characterUid?) → bool / void
Unlock(entityUid, ownerService?) → void
IsLocked(entityUid, requiredScope?) → bool
```

### Lock

Добавляет uid в реестр.  
Если uid уже занят конфликтующим scope — отказ (или политика «не перезаписывать чужой SellPending / Drop»).

Типичные вызовы:

- EntityManager при flush пачки с `Drop` → `Lock(uid, Drop, InventoryOps, EntityManager)` (на время in-flight);
- Trade при старте sell → `Lock(uid, SellPending, InventoryOps, TradeManager)`.

### Unlock

Снимает запись. Вызывать после завершения операции владельца lock:

- EntityManager: Ok/Fail enqueue-операции;
- Trade: Fail sell; после успешного sell — когда сущность уже не нужна в мире / после `removeEntity`;
- всегда после CommandBus `removeEntity` и hard-reset анализатора дюпов ([[EntityManager.DupeAnalyzer.md]]), даже если lock ставил другой сервис.

### IsLocked

`true`, если uid в реестре и scope пересекается с запрашиваемым.

- Trade перед sell: если `IsLocked` — **не** отправлять продажу на бекенд.
- EntityManager перед `enqueue*`: если уже `SellPending` — не выбрасывать / не передавать предмет.
- Подбор вторым игроком: если uid locked (in-flight / SellPending) — отказ; если ops только в очереди — подбор допустим и встаёт в очередь uid.

---

## Области блокировки (scope)

| Scope | Запрещает | Не запрещает |
|-------|-----------|--------------|
| `InventoryOps` | pick / drop / move / transfer / sell этого uid | операции с другими uid; езда на технике |
| `ContainerDispose` | выбросить / передать / снять контейнер как предмет | вождение машины, движение по миру |

Примеры:

- Пачка с дропом банки ушла (in-flight) → `InventoryOps` на банке; второй игрок не поднимает, Trade не продаёт.
- Старт sell (Trade) → `SellPending` + `InventoryOps`; EntityManager не примет `enqueueDropEntity` / `enqueueTransferEntity` по этой банке.
- Банка → в рюкзак → на in-flight дополнительно `ContainerDispose` на рюкзаке (нельзя выбросить рюкзак, пока банка in-flight).
- Банка → в машину → lock только банки на in-flight; машину для езды не блокировать.

---

## Сценарий: продажа

```text
1. Trade: IsLocked(банка)? 
   да → отказать UI / подождать
   нет → Lock(банка, SellPending, InventoryOps, TradeManager)
2. Локально банку можно изъять из доступного инвентаря (политика игры)
3. Trade → бекенд sell
4a. Fail → Unlock(банка); вернуть предмет в доступность
4b. Ok → бекенд ставит CommandBus removeEntity
       → EntityManager удаляет сущность (игнорируя lock)
       → Unlock(банка)  // EntityManager или общий хук после remove
```

Пока sell in-flight, игрок **не может** выбросить банку: `enqueueDropEntity` видит `IsLocked` и отказывает.

---

## Сценарий: дроп / move (EntityManager)

```text
1. EntityManager: IsLocked? (in-flight / SellPending → отказ)
2. оптимизм в мире + запись в очередь (без Lock)
3. flush (таймер / barrier) → Lock(..., EntityManager) → пачка на бекенд
4. ответ бекенда → Unlock (EntityManager)
```

Если Trade спросит `IsLocked` во время in-flight — получит `true`, sell не стартует. Пока ops только в очереди — `IsLocked` от EM нет; перед sell Trade делает barrier/flush при необходимости (см. [[EntityManager.Operations.md]]).

---

## Анти-паттерны

1. Дублировать свой lock внутри Trade и внутри EntityManager вместо общего сервиса.
2. Trade не вызывает `Lock` на время sell — игрок успевает сделать `enqueueDropEntity`.
3. EntityManager игнорирует `SellPending` и выбрасывает продаваемый предмет.
4. Отказать CommandBus `removeEntity` из‑за записи в реестре.
5. Глобальный lock на весь мир.
6. Блокировать вождение машины из‑за предмета в её инвентаре.
7. Держать lock EM на uid всё время, пока ops лежат в очереди до flush (мешает Drop→PickUp в одной пачке).

---

## Связанные документы

- [[EntityManager.md]] — реестр сущностей; Lock при flush / in-flight
- [[EntityManager.Operations.md]] — жизненный цикл операций
- [[EntityManager.DupeAnalyzer.md]] — hard-reset снимает lock
- [[TradeManager.md]] — sell и блокировка предмета
- [[BackendGameMutation.md]] — removeEntity после sell
- [[Architecture.md]] — оглавление
