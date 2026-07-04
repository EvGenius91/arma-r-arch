## Методы покупки TradeManager

Назад: [[TradeManager.md]]
Сущности: [[TradeManager.Entities.md]]

Документ описывает публичное поведение JSON-RPC API.

### Пересчёт состояния корзины

TradeManager возвращает актуальный снимок корзины в поле `card` публичных API-ответов. В этом снимке пересчитаны `buyFailReasons`, `moneyShortfall`, `totalSum` у [[TradeManager.Entities.md#BuyCard]] и `buyFailReasons` у каждой [[TradeManager.Entities.md#BuyCardPosition]].

Снимок возвращается в ответах:

- [[#getActiveBuyCard]]
- [[#createBuyCard]]
- [[#recreateBuyCard]]

После мутаций, которые не возвращают `card`, клиент получает актуальное состояние через повторный вызов [[#getActiveBuyCard]].

**Cart-level проверки** (в `BuyCard.buyFailReasons`):
- совместимость `BuyCardTypeEnum` и `DeliveryMethod`;
- `EmptyBuyCard` — если `positions` пуст;
- `CannotAfford` — `totalSum` превышает деньги персонажа; при этом `moneyShortfall = max(0, totalSum − деньги персонажа)`, иначе `moneyShortfall = 0`;
- `CannotFitInInventory` — для `DeliveryMethod.Inventory`: недостаточно места в инвентаре для всей корзины (без указания позиции);
- `CannotSpawnVehicleNoFreeSlot` — для `DeliveryMethod.NearVehicleSpawnPosition`: нет свободного слота спавна техники.

**Position-level проверки** (в `BuyCardPosition.buyFailReasons`):
- остаток на складе (`InsufficientStock`);
- доступность предложения (`ShopProductNotAvailableForBuy`, `ShopProductNotFound`, `ShopProductNotInShop`).

**Агрегация `ErrorInPosition`:** если нативных cart-level причин нет, но хотя бы у одной позиции `buyFailReasons` не пуст — в `BuyCard.buyFailReasons` добавляется `ErrorInPosition`.

**Инвариант клиента:** `canBuy ⟺ card.buyFailReasons` пуст. Пустой `buyFailReasons` не гарантирует успех [[#buy]] (возможна гонка); после `buy = Fail` клиент обязан вызвать [[#getActiveBuyCard]].

**Refetch корзины:** клиент перезапрашивает [[#getActiveBuyCard]] после успешного [[#addToBuyCard]], после [[#changeBuyCardPositionQuantity]], после удаления позиции или корзины, после `buy = Fail`, при изменении денег/инвентаря персонажа и по событиям с сервера.

### createBuyCard

**Сигнатура**
createBuyCard(string shopUid, string characterUid, [[TradeManager.Entities.md#BuyCardTypeEnum]] type, [[TradeManager.Entities.md#DeliveryMethod]] deliveryMethod): [[TradeManager.Entities.md#CreateBuyCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `type: BuyCardTypeEnum` - тип создаваемой корзины покупки.
- `deliveryMethod: DeliveryMethod` - метод доставки для корзины.

**Описание**
Создаёт [[TradeManager.Entities.md#BuyCard]] для персонажа в указанном магазине с выбранным типом и методом доставки. В `card` пересчитаны `buyFailReasons` (для пустой корзины — `EmptyBuyCard`), `moneyShortfall = 0`, `totalSum = 0`.

**Проверки**
- У персонажа не должно быть другой активной корзины покупки.
- `type` и `deliveryMethod` должны быть совместимы.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
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
Создаёт [[TradeManager.Entities.md#BuyCard]] для персонажа в указанном магазине с выбранным типом и методом доставки. Если у персонажа уже есть активная корзина покупки, она удаляется вместе со всеми позициями; возвращается **новая** корзина с новым `uid` и пустым `positions`. Клиент обязан использовать `card.uid` из ответа для последующих вызовов (`addToBuyCard`, `buy` и т.д.); прежний `buyCardUid` после успешного вызова недействителен. В `card` пересчитаны `buyFailReasons` (для пустой корзины — `EmptyBuyCard`), `moneyShortfall = 0`, `totalSum = 0`.

Для первого создания корзины без сброса существующей используйте [[#createBuyCard]].

**Проверки**
- `type` и `deliveryMethod` должны быть совместимы.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
- `CannotUseDeliveryMethodForBuyCardType`

### getActiveBuyCard

**Сигнатура**
getActiveBuyCard(string characterUid): [[TradeManager.Entities.md#GetActiveBuyCardResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.

**Описание**
Возвращает текущую активную [[TradeManager.Entities.md#BuyCard]] персонажа с пересчитанными `buyFailReasons`, `moneyShortfall`, `totalSum`. В `card.positions` — позиции [[TradeManager.Entities.md#BuyCardPosition]] (`uid`, `shopProductUid`, `name`, `prefabName`, `quantity`, `lineSum`, `buyFailReasons`) для UI.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- Если активной корзины покупки нет: `status = Fail` и `failReason = NoActiveBuyCard`.

**Причины отказа (`failReason`)**
- `NoActiveBuyCard`

### addToBuyCard

**Сигнатура**
addToBuyCard(string buyCardUid, string shopProductUid, int quantity): [[TradeManager.Entities.md#AddToBuyCardResult]]

**Аргументы**
- `buyCardUid: string` - идентификатор корзины покупки.
- `shopProductUid: string` - идентификатор строки каталога в смысле поля `uid` у [[TradeManager.Entities.md#ShopProduct]] (тот же идентификатор, что приходит в `products` из [[TradeManager.ShopMethods.md#getProductsForBuyByCharacter]]).
- `quantity: int` - добавляемое количество.

**Описание**
При успехе создаётся новая [[TradeManager.Entities.md#BuyCardPosition]] с указанным `quantity`. В одной корзине не более одной позиции на `shopProductUid`; повторный вызов с тем же `shopProductUid` отклоняется — для изменения количества используется [[#changeBuyCardPositionQuantity]]. Ответ содержит `positionUid`; после успеха клиент вызывает [[#getActiveBuyCard]] для получения актуального `card`.

При нехватке товара на складе операция отклоняется целиком — частичное добавление не выполняется.

**Проверки**
- `quantity` должно быть больше 0.
- Предложение с `shopProductUid` существует в каталоге TradeManager.
- Предложение относится к тому же магазину, что и корзина (`BuyCard.shopUid`).
- Предложение в текущих условиях допускает добавление в корзину: доступно к покупке и имеет достаточный `availableQuantity`.
- Тип товара, выведенный из предложения, совместим с `BuyCard.type`.
- Предложение ещё не присутствует в корзине (нет позиции с тем же `shopProductUid`).

**Результат**
- При успехе возвращает `status = Ok`, заполняет `positionUid` ([[TradeManager.Entities.md#BuyCardPosition]] `uid` созданной позиции).
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
- `CannotAddShopProductNotFound`
- `CannotAddShopProductNotInShop`
- `CannotAddShopProductNotAvailableForBuy`
- `CannotAddShopProductAlreadyInBuyCard`
- `CannotAddIncompatibleBuyCardType`
- `CannotUseNonPositiveQuantity`

### changeBuyCardPositionQuantity

**Сигнатура**
changeBuyCardPositionQuantity(string buyCardPositionUid, int newQuantity): [[TradeManager.Entities.md#ChangeBuyCardPositionQuantityResult]]

**Аргументы**
- `buyCardPositionUid: string` - идентификатор позиции в корзине покупки.
- `newQuantity: int` - новое количество для позиции.

**Описание**
Меняет `quantity` позиции. Ответ содержит только `status` и, при отказе, `failReason`; актуальный снимок корзины клиент получает через [[#getActiveBuyCard]].

Если запрошенное `newQuantity` превышает доступный остаток на складе, `quantity` откатывается к максимально допустимому значению (не меньше 1), возвращается `status = Fail` и `failReason`. Актуальное `quantity` и причины клиент получает через [[#getActiveBuyCard]] после ответа метода.

**Проверки**
- `newQuantity` должно быть больше 0.
- Остаток на складе и доступность предложения (как у position-level `buyFailReasons`).

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
- `CannotUseNonPositiveQuantity`
- `InsufficientStock`
- `ShopProductNotAvailableForBuy`
- `ShopProductNotFound`
- `ShopProductNotInShop`

### removeBuyCardPosition

**Сигнатура**
removeBuyCardPosition(string buyCardPositionUid): void

**Аргументы**
- `buyCardPositionUid: string` - идентификатор удаляемой позиции.

**Описание**
Удаляет позицию из корзины. После вызова клиент перезапрашивает корзину через [[#getActiveBuyCard]] для обновления `buyFailReasons` и `moneyShortfall`.

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

**Описание**
Выполняет покупку. Возвращает только `status`; при `Fail` причину не указывает. Клиент вызывает [[#getActiveBuyCard]] для получения актуальных `buyFailReasons` и `buyFailReasons` позиций.

Допустимо, что `buy` вернёт `Fail`, даже если предыдущий снимок корзины показывал пустой `buyFailReasons` (гонка состояния).

**Проверки**
- Совместимость `BuyCardTypeEnum` и `DeliveryMethod`.
- Наличие достаточного количества денег: сумма к оплате = `totalSum` корзины.
- Для `DeliveryMethod.Inventory` - наличие свободного места в инвентаре.
- Для `DeliveryMethod.NearVehicleSpawnPosition` - наличие свободного слота спавна техники.
- Position-level: остаток на складе, доступность предложений по каждой позиции.

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail`.
