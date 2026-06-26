## Методы магазина TradeManager

Назад: [[TradeManager.md]]
Сущности: [[TradeManager.Entities.md]]

### getShopCategories

**Сигнатура**
getShopCategories(string shopUid): [[TradeManager.Entities.md#GetShopCategoriesResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.

**Описание**
Возвращает список категорий товаров, доступных в указанном магазине.

**Результат**
- При успехе возвращает `status = Ok` и заполненный массив `categories`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `ShopNotFound`

### getProductsForBuyByCharacter

**Сигнатура**
getProductsForBuyByCharacter(string shopUid, string categoryUid, string characterUid): [[TradeManager.Entities.md#ShopProductsResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `categoryUid: string` - идентификатор категории ([[TradeManager.Entities.md#ShopCategory]]).
- `characterUid: string` - идентификатор персонажа (контекст для цен, доступности и т.п. по правилам проекта).

**Описание**
Возвращает список товаров магазина в указанной категории с учётом персонажа. Поле `uid` у каждого элемента `products` ([[TradeManager.Entities.md#ShopProduct]]) используется при добавлении в корзину в [[TradeManager.BuyMethods.md#addToBuyCard]] как `shopProductUid`.

**Результат**
- При успехе возвращает `status = Ok` и заполненный массив `products`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `ShopNotFound`
- `CategoryNotFound`
- `CharacterNotFound`