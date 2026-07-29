Менеджер торговли. Через него проходят все операции покупки/продажи и взаимодействие со средствами игрока.

## Блокировка сущности при продаже

На время `sell` TradeManager работает с общим сервисом [[EntityLockRegistry.md|EntityLockRegistry]] (не с приватным lock EntityManager):

1. При необходимости barrier/flush через EntityManager (открытие магазина / перед sell), чтобы uid инвентаря не были pending/in-flight — см. [[EntityManager.Operations.md]].
2. `IsLocked(entityUid)` — если уже занято (пачка EM in-flight) → sell локально не начинать.
3. `Lock(entityUid, reason: SellPending, scope: InventoryOps, ownerService: TradeManager)` — игрок не сможет `enqueueDropEntity` / передать предмет, пока нет ответа бекенда.
4. Fail sell → `Unlock`.
5. Ok sell → мир чистит CommandBus `removeEntity` ([[BackendGameMutation.md]]); затем `Unlock` (после удаления сущности).

Так продажа и операции EntityManager делят один реестр блокировок.

## Структура документации

- [[TradeManager.Entities.md]]
- [[TradeManager.BuyMethods.md]]
- [[TradeManager.SellMethods.md]]
- [[TradeManager.ShopMethods.md]]
- [[EntityLockRegistry.md]] — общий реестр блокировок с EntityManager

## Быстрая навигация

### Сущности
- [[TradeManager.Entities.md#OperationStatusEnum]]
- [[TradeManager.Entities.md#BuyCard]]
- [[TradeManager.Entities.md#BuyCardTypeEnum]]
- [[TradeManager.Entities.md#BuyCardPosition]]
- [[TradeManager.Entities.md#BuyCardFailReasonEnum]]
- [[TradeManager.Entities.md#BuyCardPositionFailReasonEnum]]
- [[TradeManager.Entities.md#DeliveryMethod]]
- [[TradeManager.Entities.md#AddToBuyCardResult]]
- [[TradeManager.Entities.md#AddToBuyCardFailReasonEnum]]
- [[TradeManager.Entities.md#ChangeBuyCardPositionQuantityResult]]
- [[TradeManager.Entities.md#BuyStatus]]
- [[TradeManager.Entities.md#CreateBuyCardResult]]
- [[TradeManager.Entities.md#RecreateBuyCardResult]]
- [[TradeManager.Entities.md#GetActiveBuyCardResult]]
- [[TradeManager.Entities.md#SellCard]]
- [[TradeManager.Entities.md#SellCardPosition]]
- [[TradeManager.Entities.md#SellCardFailReasonEnum]]
- [[TradeManager.Entities.md#SellCardPositionFailReasonEnum]]
- [[TradeManager.Entities.md#CreateSellCardResult]]
- [[TradeManager.Entities.md#RecreateSellCardResult]]
- [[TradeManager.Entities.md#GetActiveSellCardResult]]
- [[TradeManager.Entities.md#AddToSellCardResult]]
- [[TradeManager.Entities.md#ChangeSellCardPositionQuantityResult]]
- [[TradeManager.Entities.md#SellResult]]
- [[TradeManager.Entities.md#GetInventoryForSellResult]]
- [[TradeManager.Entities.md#GetInventoryForSellFailReasonEnum]]
- [[TradeManager.Entities.md#InventoryForSell]]
- [[TradeManager.Entities.md#InventoryProductPosition]]
- [[TradeManager.Entities.md#ShopCategory]]
- [[TradeManager.Entities.md#GetShopCategoriesResult]]
- [[TradeManager.Entities.md#ShopProduct]]
- [[TradeManager.Entities.md#ShopProductsResult]]
- [[TradeManager.Entities.md#GetDescriptionByPrefabResult]]
- [[TradeManager.Entities.md#GetDescriptionByPrefabFailReasonEnum]]

### Методы покупки
- [[TradeManager.BuyMethods.md#createBuyCard]]
- [[TradeManager.BuyMethods.md#recreateBuyCard]]
- [[TradeManager.BuyMethods.md#getActiveBuyCard]]
- [[TradeManager.BuyMethods.md#addToBuyCard]]
- [[TradeManager.BuyMethods.md#changeBuyCardPositionQuantity]]
- [[TradeManager.BuyMethods.md#removeBuyCardPosition]]
- [[TradeManager.BuyMethods.md#removeBuyCard]]
- [[TradeManager.BuyMethods.md#buy]]

### Методы продажи
- [[TradeManager.SellMethods.md#createSellCard]]
- [[TradeManager.SellMethods.md#recreateSellCard]]
- [[TradeManager.SellMethods.md#getActiveSellCard]]
- [[TradeManager.SellMethods.md#getInventoryForSell]]
- [[TradeManager.SellMethods.md#addToSellCard]]
- [[TradeManager.SellMethods.md#changeSellCardPositionQuantity]]
- [[TradeManager.SellMethods.md#removeSellCardPosition]]
- [[TradeManager.SellMethods.md#removeSellCard]]
- [[TradeManager.SellMethods.md#sell]]

### Методы магазина
- [[TradeManager.ShopMethods.md#getShopCategories]]
- [[TradeManager.ShopMethods.md#getProductsForBuyByCharacter]]
- [[TradeManager.ShopMethods.md#getDescriptionByPrefab]]
