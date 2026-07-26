# EntityLockRegistry

Назад: [[Architecture.md]]

Отдельный сервис на игровом сервере: **реестр локальных блокировок сущностей**. Не часть EntityManager и не часть TradeManager — самостоятельный компонент, от которого они зависят.

## Зависимости

```text
EntityManager  ──depends on──►  EntityLockRegistry
TradeManager   ──depends on──►  EntityLockRegistry
```

- [[EntityManager.md|EntityManager]] при каждом `enqueuePickupEntity` / `enqueueDropEntity` / `enqueueMoveEntity` (и аналогах) **обязан** поставить сущность в этот реестр через `Lock` и снять через `Unlock` после ответа бекенда.
- [[TradeManager.md|TradeManager]] на время продажи ставит `SellPending` в тот же реестр, чтобы EntityManager не принял дроп/передачу продаваемого uid.
- Реестр **не** знает о содержимом контейнеров и не ходит на бекенд — только занятость `entityUid`.

## Назначение

Пока сущность занята (операция EntityManager ещё не подтверждена бекендом, либо идёт продажа через Trade), повторный dispose локально запрещён:

- EntityManager не даст выбросить / переложить / передать уже продаваемую банку;
- Trade не начнёт `sell`, если банка уже в lock после дропа / передачи;
- второй игрок не поднимет банку, пока она в реестре после дропа первого.

Реестр **не** источник истины бекенда и **не** отменяет CommandBus: `removeEntity` с бекенда выполняется даже для заблокированного uid; после удаления запись из реестра снимается.

---

## Связь с enqueue EntityManager

При вызове методов постановки в очередь EntityManager всегда проходит через EntityLockRegistry:

```text
EntityManager.enqueuePickupEntity / enqueueDropEntity / enqueueMoveEntity
  → EntityLockRegistry.IsLocked(entityUid)
  → EntityLockRegistry.Lock(entityUid, reason, scope, ownerService: EntityManager)
  → локальный оптимизм + запись в очередь EntityManager
  → отправка на бекенд
  → Ok / Fail
  → EntityLockRegistry.Unlock(entityUid)
```

Без успешного `Lock` операция не считается принятой в синк (отказ или ожидание очереди по политике EntityManager). Подробные сигнатуры enqueue — в [[EntityManager.md#enqueuedropentity|EntityManager.md]].

---

## Кто вызывает

| Сервис | Когда |
|--------|--------|
| [[EntityManager.md\|EntityManager]] | каждый `enqueueDropEntity` / `enqueuePickupEntity` / `enqueueMoveEntity` / … — `Lock` при постановке в очередь; `Unlock` после Ok/Fail; очистка при hard-reset / `removeEntity` |
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

- EntityManager при `enqueueDropEntity` → `Lock(uid, Drop, InventoryOps, EntityManager)`;
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
- Подбор вторым игроком: если uid locked — отказ.

---

## Области блокировки (scope)

| Scope | Запрещает | Не запрещает |
|-------|-----------|--------------|
| `InventoryOps` | pick / drop / move / transfer / sell этого uid | операции с другими uid; езда на технике |
| `ContainerDispose` | выбросить / передать / снять контейнер как предмет | вождение машины, движение по миру |

Примеры:

- Дроп банки (EntityManager) → `InventoryOps` на банке; второй игрок не поднимает, Trade не продаёт.
- Старт sell (Trade) → `SellPending` + `InventoryOps`; EntityManager не примет `enqueueDropEntity` / `enqueueTransferEntity` по этой банке.
- Банка → в рюкзак → дополнительно `ContainerDispose` на рюкзаке (нельзя выбросить рюкзак, пока банка in-flight).
- Банка → в машину → lock только банки; машину для езды не блокировать.

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
1. EntityManager: IsLocked?
2. Lock(..., EntityManager)
3. enqueue в очередь EntityManager + оптимизм в мире
4. ответ бекенда → Unlock (EntityManager)
```

Если в это время Trade спросит `IsLocked` — получит `true`, sell не стартует.

---

## Анти-паттерны

1. Дублировать свой lock внутри Trade и внутри EntityManager вместо общего сервиса.
2. Trade не вызывает `Lock` на время sell — игрок успевает сделать `enqueueDropEntity`.
3. EntityManager игнорирует `SellPending` и выбрасывает продаваемый предмет.
4. Отказать CommandBus `removeEntity` из‑за записи в реестре.
5. Глобальный lock на весь мир.
6. Блокировать вождение машины из‑за предмета в её инвентаре.

---

## Связанные документы

- [[EntityManager.md]] — реестр сущностей; клиент Lock/Unlock при enqueue*
- [[EntityManager.Operations.md]] — жизненный цикл операций
- [[EntityManager.DupeAnalyzer.md]] — hard-reset снимает lock
- [[TradeManager.md]] — sell и блокировка предмета
- [[BackendGameMutation.md]] — removeEntity после sell
- [[Architecture.md]] — оглавление
