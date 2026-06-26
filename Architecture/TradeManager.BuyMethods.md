## Методы покупки TradeManager

Назад: [[TradeManager.md]]
Сущности: [[TradeManager.Entities.md]]

### createBuyCard

**Сигнатура**
createBuyCard(string shopUid, string characterUid, [[TradeManager.Entities.md#BuyCardTypeEnum]] type, [[TradeManager.Entities.md#DeliveryMethod]] deliveryMethod): [[TradeManager.Entities.md#CreateBuyCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `type: BuyCardTypeEnum` - тип создаваемой корзины покупки.
- `deliveryMethod: DeliveryMethod` - метод доставки для корзины.

**Описание**
Создаёт [[TradeManager.Entities.md#BuyCard]] для персонажа в указанном магазине с выбранным типом и методом доставки.

**Проверки**
- У персонажа не должно быть другой активной корзины покупки.
- `type` и `deliveryMethod` должны быть совместимы.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CannotCreateBuyCardAlreadyExists`
- `CannotUseDeliveryMethodForBuyCardType`

### recreateBuyCard

**Сигнатура**
recreateBuyCard(string shopUid, string characterUid, [[TradeManager.Entities.md#BuyCardTypeEnum]] type, [[TradeManager.Entities.md#DeliveryMethod]] deliveryMethod): [[TradeManager.Entities.md#RecreateBuyCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `type: BuyCardTypeEnum` - тип создаваемой корзины покупки.
- `deliveryMethod: DeliveryMethod` - метод доставки для корзины.

**Описание**
Создаёт [[TradeManager.Entities.md#BuyCard]] для персонажа в указанном магазине с выбранным типом и методом доставки. Если у персонажа уже есть активная корзина покупки, она удаляется вместе со всеми позициями; возвращается **новая** корзина с новым `uid` и пустым `positions`. Клиент обязан использовать `card.uid` из ответа для последующих вызовов (`addToBuyCard`, `canBuyCard`, `buy` и т.д.); прежний `buyCardUid` после успешного вызова недействителен.

Для первого создания корзины без сброса существующей используйте [[#createBuyCard]].

**Проверки**
- `type` и `deliveryMethod` должны быть совместимы.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CannotUseDeliveryMethodForBuyCardType`

### getActiveBuyCard

**Сигнатура**
getActiveBuyCard(string characterUid): [[TradeManager.Entities.md#GetActiveBuyCardResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.

**Описание**
Возвращает текущую активную [[TradeManager.Entities.md#BuyCard]] персонажа, если она есть. В `card.positions` — позиции [[TradeManager.Entities.md#BuyCardPosition]] (`uid`, `shopProductUid`, `name`, `prefabName`, `quantity`, `lineSum`) для UI.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- Если активной корзины покупки нет: `status = Fail` и `failReason = NoActiveBuyCard`.

**Ошибки / причины отказа**
- `NoActiveBuyCard`

### addToBuyCard

**Сигнатура**
addToBuyCard(string buyCardUid, string shopProductUid, int quantity): [[TradeManager.Entities.md#AddToBuyCardResult]]

**Аргументы**
- `buyCardUid: string` - идентификатор корзины покупки.
- `shopProductUid: string` - идентификатор строки каталога в смысле поля `uid` у [[TradeManager.Entities.md#ShopProduct]] (тот же идентификатор, что приходит в `products` из [[TradeManager.ShopMethods.md#getProductsForBuyByCharacter]]). Префаб, тип товара для [[TradeManager.Entities.md#BuyCardTypeEnum]] и цена для последующих проверок разрешаются в TradeManager по этому uid и по магазину корзины (`shopUid` у [[TradeManager.Entities.md#BuyCard]]).
- `quantity: int` - добавляемое количество.

**Описание**
При успехе создаётся новая [[TradeManager.Entities.md#BuyCardPosition]] или увеличивается `quantity` уже существующей позиции с тем же `shopProductUid` (в одной корзине не более одной позиции на uid предложения). TradeManager заполняет `name` и `prefabName` из предложения и пересчитывает `lineSum` по актуальной цене предложения и итоговому `quantity`.

**Проверки**
- `quantity` должно быть больше 0.
- Предложение с `shopProductUid` существует в каталоге TradeManager.
- Предложение относится к тому же магазину, что и корзина (`BuyCard.shopUid`).
- Предложение в текущих условиях допускает добавление в корзину (по правилам проекта: доступность к покупке, остаток и т.п., в духе [[TradeManager.Entities.md#ShopProduct]] `isAvailableForBuy` / `availableQuantity`).
- Тип товара, выведенный из предложения, совместим с `BuyCard.type`.

**Результат**
- При успехе возвращает `status = Ok`, заполняет `positionUid` ([[TradeManager.Entities.md#BuyCardPosition]] `uid` созданной или обновлённой позиции).
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CannotAddShopProductNotFound`
- `CannotAddShopProductNotInShop`
- `CannotAddShopProductNotAvailableForBuy`
- `CannotAddIncompatibleBuyCardType`
- `CannotUseNonPositiveQuantity`

### changeBuyCardPositionQuantity

**Сигнатура**
changeBuyCardPositionQuantity(string buyCardPositionUid, int newQuantity): [[TradeManager.Entities.md#ChangeBuyCardPositionQuantityResult]]

**Аргументы**
- `buyCardPositionUid: string` - идентификатор позиции в корзине покупки.
- `newQuantity: int` - новое количество для позиции.

**Описание**
После смены `quantity` TradeManager пересчитывает `lineSum` позиции по актуальной цене предложения.

**Проверки**
- `newQuantity` должно быть больше 0.

**Ошибки / причины отказа**
- `CannotUseNonPositiveQuantity`

### removeBuyCardPosition

**Сигнатура**
removeBuyCardPosition(string buyCardPositionUid): void

**Аргументы**
- `buyCardPositionUid: string` - идентификатор удаляемой позиции.

### canBuyCard

**Сигнатура**
canBuyCard(string buyCardUid): [[TradeManager.Entities.md#CanBuyStatus]]

**Аргументы**
- `buyCardUid: string` - идентификатор корзины покупки.

**Проверки**
- Совместимость `BuyCardTypeEnum` и `DeliveryMethod`.
- Наличие достаточного количества денег: сумма к оплате = сумма `lineSum` по всем позициям корзины.
- Для `DeliveryMethod.Inventory` - наличие свободного места в инвентаре.
- Для `DeliveryMethod.NearVehicleSpawnPosition` - наличие свободного слота спавна техники.

### removeBuyCard

**Сигнатура**
removeBuyCard(string buyCardUid): void

**Аргументы**
- `buyCardUid: string` - идентификатор корзины покупки.

### buy

**Сигнатура**
buy(string buyCardUid): [[TradeManager.Entities.md#BuyStatus]]

**Аргументы**
- `buyCardUid: string` - идентификатор корзины покупки.

**Проверки**
- Совместимость `BuyCardTypeEnum` и `DeliveryMethod`.
- Наличие достаточного количества денег: сумма к оплате = сумма `lineSum` по всем позициям корзины.
- Для `DeliveryMethod.Inventory` - наличие свободного места в инвентаре.
- Для `DeliveryMethod.NearVehicleSpawnPosition` - наличие свободного слота спавна техники.

**Ошибки / причины отказа**
- `CannotAfford`
- `CannotFitInInventory`
- `CannotSpawnVehicleNoFreeSlot`
- `CannotUseDeliveryMethodForBuyCardType`
