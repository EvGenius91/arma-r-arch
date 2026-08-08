# Архитектура

Оглавление документации менеджеров игрового сервера.

## Бекенд (Laravel)

- [[../ArchBackend/ArchBackend.md]] — контракты сервисов, разделение Service и API

## Принципы

- [[BackendGameMutation.md]] — как бекенд принимает решение, а CommandBus применяет его в игровом мире (продажа, покупка)
- [[../Common concepts.md]] — GameBackendManager и правило: все события с бекенда → Local EventBus

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

## EntityManager

Реестр сущностей мира (техника, предметы инвентаря / экипировки). Предмет остаётся сущностью; к инвентарю привязан контейнером и слотом. Синхронизация владения и расположения с бекендом — зона EntityManager, не CharacterStateManager. При спавне / респавне заполняет инвентарь нового персонажа через `loadInventory`. **Зависит от EntityLockRegistry.**

- [[EntityManager.md]] — реестр, зависимость от EntityLockRegistry, `loadInventory`, enqueue*
- [[EntityManager.Entities.md]] — `EntityItem`, `StorageTypeEnum`, `GetInventoryByCharacterUidResult`, `FindEntitiesByUidListResult`
- [[EntityManager.Operations.md]] — очередь, flush (~1 с / barrier), порядок пачки, lock на in-flight
- [[EntityManager.HttpMethods.md]] — JSON-RPC `entity@getInventoryByCharacterUid`, `entity@findEntitiesByUidList`, `entity@applyOperations`
- [[EntityManager.DupeAnalyzer.md]] — анализатор дюпов, hard-reset, `resetGeneration`, игнор устаревших операций

## CharacterStateManager

Синхронизация состояния персонажа (здоровье, витальность, позиция, проекции). Инвентарь в снимке — проекция из EntityManager.

- [[CharacterStateManager (черновик) 3.md]] — операции персонажа, `applyOperations`, recovery
