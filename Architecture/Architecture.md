# Архитектура

Оглавление документации менеджеров игрового сервера.

## Бекенд (Laravel)

- [[../ArchBackend/ArchBackend.md]] — контракты сервисов, разделение Service и API

## Принципы

- [[BackendGameMutation.md]] — как бекенд принимает решение, а CommandBus применяет его в игровом мире (продажа, покупка)
- [[../Common concepts.md]] — GameBackendManager и правило: все события с бекенда → Local EventBus

## GameManager

Жизненный цикл игровой сессии: регистрация `ServerSession` при старте мира (`game@gameStarted`).

- [[GameManager.md]] — обзор и метод `gameStarted`
- [[GameManager.Entities.md]] — `ServerSession`, `GameStartedResult`

## CommandBus

Доставка команд бекенд → игровой сервер: опрос каждые 0.5 с, маршрутизация в целевые сервисы, отчёт `Completed` / `Fail`.

- [[../CommandBus/CommandBus.md]] — `run`, `reportCommands`
- [[../CommandBus/CommandBus.HttpMethods.md]] — JSON-RPC `command@getPendingCommands`, `command@reportCommands`
- [[../CommandBus/CommandBus.Entities.md]] — `Command`, `CommandReport`, Result DTO
- [[../CommandBus/Commands.md]] — каталог команд (`SpawnEntity`, `DeleteEntity`, `ParsePrefab`, `SetLockEntity`, `ChangeLockStatus`, `AddSafeZone`, `SetEntityOwner`, `SetAccessByType`)

## TradeManager

Менеджер торговли: покупка, продажа, взаимодействие со средствами игрока.

- [[TradeManager.md]] — обзор и быстрая навигация
- [[TradeManager.Entities.md]] — сущности и enum
- [[TradeManager.BuyMethods.md]] — методы покупки
- [[TradeManager.SellMethods.md]] — методы продажи
- [[TradeManager.ShopMethods.md]] — методы магазина

## BankingManager

Менеджер банковских операций и денежных средств персонажа.

- [[BankingManager.md]] — обзор и методы
- [[BankingManager.Entities.md]] — сущности и enum

## EntityLockRegistry

Отдельный сервис локальных блокировок `entityUid`. От него зависят EntityManager и TradeManager.

- [[EntityLockRegistry.md]] — API Lock / Unlock / IsLocked; EntityManager блокирует uid на время in-flight пачки; Trade — SellPending при продаже

## SafeZoneService

Игровой сервис списка safe-zone и правил доступа по `ownerUidList`. В зоне доступ к сущности с непустым `ownerUidList` имеют только эти персонажи, независимо от замка. Бекенд хранит зоны; в мир они попадают командой CommandBus `AddSafeZone`.

- [[SafeZoneService.md]] — `TSM_SafeZoneService.isInSafeZone`, `TSM_EntitySafeZone.isAccess`, `TSM_EntityProps.ownerUidList`
- [[SafeZoneService.Entities.md]] — `TSM_SafeZone`

## AccessService

Игровой сервис карты разрешений на действия над сущностью (`GetIn`, инвентарь, замок). Бекенд — источник истины; карта заполняется только командой CommandBus `SetAccessByType`. Чтение — `IsCan` (клиент через `TSM_AccessComponent`).

- [[AccessService.md]] — `TSM_AccessService.IsCan`, приём `SetAccessByType`
- [[AccessService.Entities.md]] — `AccessPermissionEnum`, грант

## VehicleService

Доменный сервис техники на бекенде: поиск по последнему водителю и/или радиусу в мире. Не реестр сущностей — положение, владение и HitZone техники хранит EntityManager; `entityUid` = `EntityItem.uid`.

- [[VehicleService.md]] — обзор, `findVehicle`, `applyOperations`, `registerVehicle`
- [[VehicleService.Entities.md]] — `Vehicle`, `VehicleFilter`, `PositionRadiusFilter`, `VehicleOperation`
- [[VehicleService.HttpMethods.md]] — JSON-RPC `vehicle@applyOperations`

## EntityManager

Реестр сущностей мира (техника, предметы инвентаря / экипировки). Предмет остаётся сущностью; к инвентарю привязан контейнером и слотом. Синхронизация владения, расположения и HitZone экземпляра с бекендом — зона EntityManager, не CharacterStateManager и не VehicleService. При спавне / респавне заполняет инвентарь нового персонажа через `loadInventory`. **Зависит от EntityLockRegistry.**

- [[EntityManager.md]] — реестр, зависимость от EntityLockRegistry, `loadInventory`, `enqueueCommand`, enqueue*
- [[EntityManager.Entities.md]] — `EntityItem`, `EntityHitZone`, `StorageTypeEnum`, `GetInventoryByCharacterUidResult`, `FindEntitiesByUidListResult`
- [[EntityManager.Operations.md]] — очередь, flush (~1 с / barrier), порядок пачки, lock на in-flight
- [[EntityManager.HttpMethods.md]] — JSON-RPC `entity@getInventoryByCharacterUid`, `entity@findEntitiesByUidList`, `entity@applyOperations`
- [[EntityManager.DupeAnalyzer.md]] — анализатор дюпов, hard-reset, `resetGeneration`, игнор устаревших операций

## CharacterStateManager (на согласовании)

Синхронизация состояния персонажа (здоровье, витальность, позиция, проекции). Инвентарь в снимке — проекция из EntityManager. Описание черновое: сервис на согласовании.

- [[CharacterStateManager (черновик) 3.md]] — операции персонажа, `applyOperations`, recovery

## LootSpawnManager (на согласовании)

Спавн лута у лутабельных объектов; регистрация сущностей через EntityManager, появление в мире через CommandBus `SpawnEntity`. Контракт сервиса на согласовании.

- упоминание потока: [[EntityManager.md]] (секция «Спавн лута»)
