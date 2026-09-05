# LockManager

Назад: [[Architecture.md]] · [[LockManager.Entities.md]]

Сервис **замков на сущностях** (дверь, контейнер и т.п.): кто может закрыть / открыть и текущее состояние `isLocked`. Не путать с [[EntityLockRegistry.md|EntityLockRegistry]] — тот реестр локальной занятости `entityUid` на время in-flight операций EntityManager / продажи Trade.

Доменный контракт бекенда: `LockService` (`Contracts/LockService/`). RPC: `lock@lock`, `lock@unlock` ([[../api/http api.md]]).

Тип замка ([[LockManager.Entities.md#LockTypeEnum]]) задаёт, чем проверяется право:

- `character` — актёр должен быть в списке ключ-персонажей;
- `keyEntity` — в инвентаре актёра должен быть ключ-сущность этого замка;
- `pinCode` — переданный пин должен совпасть с заданным в замке.

Игровой сервер вызывает RPC; бекенд фиксирует `isLocked`. Мир обновляется командой CommandBus `ChangeLockStatus` ([[../CommandBus/Commands.md#ChangeLockStatus]]). `SetLockEntity` в эти методы не ставится — замок на сущность вешается при спавне ([[GameManager.md#gameStarted]], [[../CommandBus/Commands.md#SetLockEntity]]).

В [[SafeZoneService.md|safe-zone]] доступ по `ownerUidList` важнее замка: слой `TSM_EntitySafeZone` не смотрит на `isLocked`. Замок проверяется только если `isAccess` уже `true`.

## Структура документации

- [[LockManager.Entities.md]] — `LockTypeEnum`, `LockResult`, `UnlockResult`, `LockFailReasonEnum`

## Методы

### lock

**Сигнатура**
lock(string lockUid, string|null actorCharacterUid, string|null pinCode): [[LockManager.Entities.md#LockResult]]

**Аргументы**
- `lockUid: string` — идентификатор замка.
- `actorCharacterUid: string | null` — персонаж, который закрывает замок. Обязателен для типов `character` и `keyEntity`.
- `pinCode: string | null` — пин-код. Обязателен для типа `pinCode`.

**Описание**
Закрывает замок. Бекенд проверяет право по типу замка и выставляет `isLocked = true`. Если замок уже закрыт — успех (идемпотентно), команда `ChangeLockStatus` не ставится. Если статус сменился — в очередь CommandBus ставится `ChangeLockStatus` с `isLocked = true`.

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `LockNotFound`
- `ActorRequired`
- `KeyNotFoundInInventory`
- `ActorNotInCharacterKeys`
- `PinCodeMismatch`

### unlock

**Сигнатура**
unlock(string lockUid, string|null actorCharacterUid, string|null pinCode): [[LockManager.Entities.md#UnlockResult]]

**Аргументы**
- `lockUid: string` — идентификатор замка.
- `actorCharacterUid: string | null` — персонаж, который открывает замок. Обязателен для типов `character` и `keyEntity`.
- `pinCode: string | null` — пин-код. Обязателен для типа `pinCode`.

**Описание**
Открывает замок. Бекенд проверяет право по типу замка и выставляет `isLocked = false`. Если замок уже открыт — успех (идемпотентно), команда `ChangeLockStatus` не ставится. Если статус сменился — в очередь CommandBus ставится `ChangeLockStatus` с `isLocked = false`.

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `LockNotFound`
- `ActorRequired`
- `KeyNotFoundInInventory`
- `ActorNotInCharacterKeys`
- `PinCodeMismatch`
