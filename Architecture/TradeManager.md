Менеджер торговли. Через него проходят все операции покупки/продажи и взаимодействие со средствами игрока.

## Структура документации

- [[TradeManager.Entities.md]]
- [[TradeManager.BuyMethods.md]]
- [[TradeManager.SellMethods.md]]
- [[TradeManager.ShopMethods.md]]

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
