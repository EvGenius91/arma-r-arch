## Сущности TradeManager

Назад: [[TradeManager.md]]

Сущности описаны в виде JSON-объектов и строковых enum, которые возвращает публичный JSON-RPC API.

### OperationStatusEnum

**Сигнатура**
```
enum OperationStatusEnum
{
	case Ok;
	case Fail;
}
```

**Назначение**
Общий статус операций TradeManager с исходом «успех / отказ».

**Значения**
- `Ok` - операция завершилась успешно.
- `Fail` - операция отклонена.

**Используется в**
[[#AddToBuyCardResult]], [[#ChangeBuyCardPositionQuantityResult]], [[#BuyStatus]], [[#CreateBuyCardResult]], [[#RecreateBuyCardResult]], [[#GetActiveBuyCardResult]], [[#CreateSellCardResult]], [[#RecreateSellCardResult]], [[#GetActiveSellCardResult]], [[#AddToSellCardResult]], [[#ChangeSellCardPositionQuantityResult]], [[#SellResult]], [[#GetShopCategoriesResult]], [[#ShopProductsResult]], [[#GetDescriptionByPrefabResult]], [[#GetInventoryForSellResult]]

### BuyCard

**Сигнатура**
```
class BuyCard
{
	string uid;
	string shopUid;
	string characterUid;
	BuyCardPosition[] positions;
	DeliveryMethod deliveryMethod;
	BuyCardTypeEnum type;
	BuyCardFailReasonEnum[] buyFailReasons;
	int moneyShortfall;
	int totalSum;
}
```

**Назначение**
Корзина покупки для одного персонажа. Одновременно у персонажа может существовать только одна корзина. Поля `buyFailReasons`, `moneyShortfall` и `totalSum` пересчитываются TradeManager при каждом формировании снимка корзины ([[TradeManager.BuyMethods.md#getActiveBuyCard]], [[TradeManager.BuyMethods.md#createBuyCard]], [[TradeManager.BuyMethods.md#recreateBuyCard]], после [[TradeManager.BuyMethods.md#buy]] с `status = Fail`).

**Свойства**
- `uid: string` - уникальный идентификатор корзины.
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `positions: BuyCardPosition[]` - позиции товаров в корзине.
- `deliveryMethod: DeliveryMethod` - выбранный способ доставки.
- `type: BuyCardTypeEnum` - тип корзины.
- `buyFailReasons: BuyCardFailReasonEnum[]` - причины, по которым корзину сейчас нельзя купить (cart-level). Пустой массив означает, что покупка разрешена с точки зрения корзины (`canBuy`). Не гарантирует успех [[TradeManager.BuyMethods.md#buy]] из‑за возможной гонки состояния.
- `moneyShortfall: int` - на сколько не хватает денег для покупки всей корзины: `max(0, totalSum − деньги персонажа)`. Заполняется только при наличии `CannotAfford` в `buyFailReasons`; иначе `0`.
- `totalSum: int` - сумма `lineSum` по всем позициям в `positions` (включая позиции с непустыми `buyFailReasons`); при пустом `positions` — `0`. Инвариант: `totalSum` равен сумме `lineSum` позиций в том же снимке корзины.

**Инварианты**
- На персонажа разрешена только одна активная корзина.
- `deliveryMethod` должен быть совместим с `type`.
- Среди позиций в `positions` значение `shopProductUid` не повторяется: для одного предложения каталога одна строка корзины, повторное добавление отклоняется.
- `canBuy ⟺ buyFailReasons` пуст (клиентская семантика кнопки «Купить»).
- `ErrorInPosition` в `buyFailReasons` добавляется производно: если нет нативных cart-level причин, но хотя бы у одной позиции `buyFailReasons` не пуст (см. [[#BuyCardFailReasonEnum]]).
- При пустом `positions` в `buyFailReasons` включается `EmptyBuyCard`.

**Связано**
[[#BuyCardPosition]], [[#DeliveryMethod]], [[#BuyCardTypeEnum]], [[#BuyCardFailReasonEnum]], [[#GetActiveBuyCardResult]]

### BuyCardTypeEnum

**Сигнатура**
```
enum BuyCardTypeEnum
{
	case Vehicle;
	case Inventory;
}
```

**Назначение**
Определяет тип корзины.

**Значения**
- `Vehicle` - корзина с техникой.
- `Inventory` - корзина с элементами инвентаря.

**Используется в**
[[#BuyCard]], [[#AddToBuyCardFailReasonEnum]]

### BuyCardPosition

**Сигнатура**
```
class BuyCardPosition
{
	string uid;
	string shopProductUid;
	string name;
	string prefabName;
	int quantity;
	int lineSum;
	BuyCardPositionFailReasonEnum[] buyFailReasons;
}
```

**Назначение**
Описывает одну позицию товара в корзине, привязанную к строке каталога магазина.

**Свойства**
- `uid: string` - уникальный идентификатор позиции.
- `shopProductUid: string` - идентификатор предложения в каталоге (то же значение, что `uid` у [[#ShopProduct]]); источник истины для цены, лимитов и сопоставления с [[TradeManager.BuyMethods.md#addToBuyCard]].
- `name: string` - отображаемое имя позиции (как у [[#ShopProduct]] `name`; обычно совпадает с именем товара; согласовано с предложением по `shopProductUid`).
- `prefabName: string` - имя префаба товара (для логики доставки/спавна; согласовано с предложением по `shopProductUid`).
- `quantity: int` - количество товара в позиции.
- `lineSum: int` - сумма по строке корзины покупки с учётом `quantity` (те же денежные единицы, что у [[#ShopProduct]] `unitPrice` и поле `lineSum` у [[#SellCardPosition]]); рассчитывается TradeManager при формировании и обновлении позиции.
- `buyFailReasons: BuyCardPositionFailReasonEnum[]` - причины, по которым эту позицию сейчас нельзя купить (position-level: остаток на складе, доступность предложения и т.п.). Деньги и место в инвентаре сюда не входят — только в `buyFailReasons` у [[#BuyCard]].

**Инварианты**
- `quantity` должно быть больше 0.
- В пределах одной [[#BuyCard]] каждое `shopProductUid` встречается не более чем в одной позиции.
- При `quantity > 0` ожидается `lineSum >= 0`; при нулевой или отрицательной цене предложения — по правилам проекта (обычно такие товары не попадают в корзину через [[TradeManager.BuyMethods.md#addToBuyCard]]).

**Используется в**
[[#BuyCard]], [[#BuyCardPositionFailReasonEnum]]

### BuyCardFailReasonEnum

**Сигнатура**
```
enum BuyCardFailReasonEnum
{
	case CannotAfford;
	case CannotFitInInventory;
	case CannotSpawnVehicleNoFreeSlot;
	case CannotUseDeliveryMethodForBuyCardType;
	case EmptyBuyCard;
	case ErrorInPosition;
}
```

**Назначение**
Причины отказа в покупке на уровне корзины (`BuyCard.buyFailReasons`). Отдельный enum от [[#BuyCardPositionFailReasonEnum]].

**Значения**
- `CannotAfford` - у персонажа недостаточно денег для покупки всей корзины (`totalSum` превышает деньги персонажа).
- `CannotFitInInventory` - у персонажа недостаточно места в инвентаре для всей корзины (не указывается, по какой позиции).
- `CannotSpawnVehicleNoFreeSlot` - нет свободного слота для спавна техники.
- `CannotUseDeliveryMethodForBuyCardType` - тип доставки не совместим с типом корзины.
- `EmptyBuyCard` - в корзине нет позиций.
- `ErrorInPosition` - производная причина: нативных cart-level причин нет, но хотя бы у одной [[#BuyCardPosition]] `buyFailReasons` не пуст. Детали — в `buyFailReasons` позиций.

**Используется в**
[[#BuyCard]]

### BuyCardPositionFailReasonEnum

**Сигнатура**
```
enum BuyCardPositionFailReasonEnum
{
	case InsufficientStock;
	case ShopProductNotAvailableForBuy;
	case ShopProductNotFound;
	case ShopProductNotInShop;
}
```

**Назначение**
Причины отказа в покупке на уровне одной позиции (`BuyCardPosition.buyFailReasons`).

**Значения**
- `InsufficientStock` - запрошенное `quantity` превышает `availableQuantity` предложения ([[#ShopProduct]]).
- `ShopProductNotAvailableForBuy` - предложение недоступно к покупке (`isAvailableForBuy = false` и т.п.).
- `ShopProductNotFound` - предложение с `shopProductUid` не найдено в каталоге TradeManager.
- `ShopProductNotInShop` - предложение не относится к магазину корзины (`BuyCard.shopUid`).

**Используется в**
[[#BuyCardPosition]], [[#ChangeBuyCardPositionQuantityFailReasonEnum]]

### DeliveryMethod

**Сигнатура**
```
enum DeliveryMethod
{
	case Inventory;
	case NearVehicleSpawnPosition;
}
```

**Назначение**
Определяет способ доставки купленных товаров.

**Значения**
- `Inventory` - доставка в инвентарь персонажа.
- `NearVehicleSpawnPosition` - спавн техники на ближайшей позиции.

**Ограничения**
- Для [[#BuyCardTypeEnum]] = `Inventory` допустим только `DeliveryMethod.Inventory`.
- Для [[#BuyCardTypeEnum]] = `Vehicle` допустим только `DeliveryMethod.NearVehicleSpawnPosition`.

**Используется в**
[[#BuyCard]], [[#CreateBuyCardFailReasonEnum]], [[#BuyCardFailReasonEnum]]

### AddToBuyCardResult

**Сигнатура**
```
class AddToBuyCardResult
{
	OperationStatusEnum status;
	AddToBuyCardFailReasonEnum failReason;
	string positionUid;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#addToBuyCard]].

**Связано**
[[#OperationStatusEnum]], [[#AddToBuyCardFailReasonEnum]]

### AddToBuyCardFailReasonEnum

**Сигнатура**
```
enum AddToBuyCardFailReasonEnum
{
	case CannotAddShopProductNotFound;
	case CannotAddShopProductNotInShop;
	case CannotAddShopProductNotAvailableForBuy;
	case CannotAddShopProductAlreadyInBuyCard;
	case CannotAddIncompatibleBuyCardType;
	case CannotUseNonPositiveQuantity;
}
```

**Значения**
- `CannotAddShopProductNotFound` - в каталоге TradeManager нет предложения с указанным `shopProductUid` (неизвестный или снятый с продажи uid).
- `CannotAddShopProductNotInShop` - предложение есть, но не относится к магазину корзины (`BuyCard.shopUid`).
- `CannotAddShopProductNotAvailableForBuy` - предложение в текущем состоянии нельзя добавить в корзину (недоступно к покупке, нехватка остатка и т.п. — по правилам проекта).
- `CannotAddShopProductAlreadyInBuyCard` - предложение с указанным `shopProductUid` уже присутствует в корзине; для изменения количества используется [[TradeManager.BuyMethods.md#changeBuyCardPositionQuantity]].
- `CannotAddIncompatibleBuyCardType` - после разрешения предложения по `shopProductUid` тип товара несовместим с типом корзины.
- `CannotUseNonPositiveQuantity` - количество должно быть больше 0.

**Используется в**
[[#AddToBuyCardResult]]

### ChangeBuyCardPositionQuantityResult

**Сигнатура**
```
class ChangeBuyCardPositionQuantityResult
{
	OperationStatusEnum status;
	ChangeBuyCardPositionQuantityFailReasonEnum failReason;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#changeBuyCardPositionQuantity]].

**Связано**
[[#OperationStatusEnum]], [[#ChangeBuyCardPositionQuantityFailReasonEnum]]

### ChangeBuyCardPositionQuantityFailReasonEnum

**Сигнатура**
```
enum ChangeBuyCardPositionQuantityFailReasonEnum
{
	case CannotUseNonPositiveQuantity;
	case InsufficientStock;
	case ShopProductNotAvailableForBuy;
	case ShopProductNotFound;
	case ShopProductNotInShop;
}
```

**Назначение**
Причины отказа при изменении количества позиции корзины покупки. Значения `InsufficientStock`, `ShopProductNotAvailableForBuy`, `ShopProductNotFound`, `ShopProductNotInShop` согласованы по смыслу с [[#BuyCardPositionFailReasonEnum]].

**Значения**
- `CannotUseNonPositiveQuantity` - новое количество должно быть больше 0.
- `InsufficientStock` - запрошенное `newQuantity` превышает доступный остаток; `quantity` откатывается к максимально допустимому.
- `ShopProductNotAvailableForBuy` - предложение недоступно к покупке.
- `ShopProductNotFound` - предложение не найдено в каталоге.
- `ShopProductNotInShop` - предложение не относится к магазину корзины.

**Используется в**
[[#ChangeBuyCardPositionQuantityResult]]

### BuyStatus

**Сигнатура**
```
class BuyStatus
{
	OperationStatusEnum status;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#buy]]. При `status = Fail` причины отказа не возвращаются — клиент перезапрашивает корзину через [[TradeManager.BuyMethods.md#getActiveBuyCard]].

**Свойства**
- `status: OperationStatusEnum` - итог операции покупки.

**Связано**
[[#OperationStatusEnum]]

### CreateBuyCardResult

**Сигнатура**
```
class CreateBuyCardResult
{
	OperationStatusEnum status;
	CreateBuyCardFailReasonEnum failReason;
	BuyCard card;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#createBuyCard]].

**Связано**
[[#OperationStatusEnum]], [[#CreateBuyCardFailReasonEnum]], [[#BuyCard]]

### CreateBuyCardFailReasonEnum

**Сигнатура**
```
enum CreateBuyCardFailReasonEnum
{
	case CannotCreateBuyCardAlreadyExists;
	case CannotUseDeliveryMethodForBuyCardType;
}
```

**Назначение**
Причины отказа при создании корзины покупки.

**Значения**
- `CannotCreateBuyCardAlreadyExists` - у персонажа уже есть корзина покупки.
- `CannotUseDeliveryMethodForBuyCardType` - выбранный метод доставки несовместим с типом корзины.

**Используется в**
[[#CreateBuyCardResult]]

### RecreateBuyCardResult

**Сигнатура**
```
class RecreateBuyCardResult
{
	OperationStatusEnum status;
	RecreateBuyCardFailReasonEnum failReason;
	BuyCard card;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#recreateBuyCard]].

**Связано**
[[#OperationStatusEnum]], [[#RecreateBuyCardFailReasonEnum]], [[#BuyCard]]

### RecreateBuyCardFailReasonEnum

**Сигнатура**
```
enum RecreateBuyCardFailReasonEnum
{
	case CannotUseDeliveryMethodForBuyCardType;
}
```

**Назначение**
Причины отказа при пересоздании корзины покупки.

**Значения**
- `CannotUseDeliveryMethodForBuyCardType` - выбранный метод доставки несовместим с типом корзины.

**Используется в**
[[#RecreateBuyCardResult]]

### GetActiveBuyCardResult

**Сигнатура**
```
class GetActiveBuyCardResult
{
	OperationStatusEnum status;
	GetActiveBuyCardFailReasonEnum failReason;
	BuyCard card;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#getActiveBuyCard]].

**Свойства**
- `status: OperationStatusEnum` - найдена ли активная корзина покупки.
- `failReason: GetActiveBuyCardFailReasonEnum` - причина при `status = Fail`.
- `card: BuyCard` - при `status = Ok` содержит активную корзину; при `status = Fail` не используется.

**Связано**
[[#OperationStatusEnum]], [[#GetActiveBuyCardFailReasonEnum]], [[#BuyCard]]

### GetActiveBuyCardFailReasonEnum

**Сигнатура**
```
enum GetActiveBuyCardFailReasonEnum
{
	case NoActiveBuyCard;
}
```

**Назначение**
Причины отказа при запросе активной корзины покупки.

**Значения**
- `NoActiveBuyCard` - у персонажа нет активной корзины покупки.

**Используется в**
[[#GetActiveBuyCardResult]]

### SellCard

**Сигнатура**
```
class SellCard
{
	string uid;
	string shopUid;
	string characterUid;
	SellCardPosition[] positions;
	SellCardFailReasonEnum[] sellFailReasons;
	int totalSum;
}
```

**Назначение**
Корзина продажи для одного персонажа. Одновременно у персонажа может существовать только одна корзина. Поля `sellFailReasons` и `totalSum` пересчитываются TradeManager при каждом формировании снимка корзины ([[TradeManager.SellMethods.md#getActiveSellCard]], [[TradeManager.SellMethods.md#createSellCard]], [[TradeManager.SellMethods.md#recreateSellCard]], после [[TradeManager.SellMethods.md#sell]] с `status = Fail`).

**Свойства**
- `uid: string` - уникальный идентификатор корзины.
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `positions: SellCardPosition[]` - позиции товаров на продажу.
- `sellFailReasons: SellCardFailReasonEnum[]` - причины, по которым корзину сейчас нельзя продать (cart-level). Пустой массив означает, что продажа разрешена с точки зрения корзины (`canSell`). Не гарантирует успех [[TradeManager.SellMethods.md#sell]] из‑за возможной гонки состояния.
- `totalSum: int` - сумма `lineSum` по всем позициям в `positions` (включая позиции с непустыми `sellFailReasons`); при пустом `positions` — `0`. Инвариант: `totalSum` равен сумме `lineSum` позиций в том же снимке корзины.

**Инварианты**
- На персонажа разрешена только одна активная корзина продажи.
- Каждый `entityUid` из массивов `entityUids` всех позиций встречается в корзине не более одного раза (без повторов между строками и внутри списков).
- `canSell ⟺ sellFailReasons` пуст (клиентская семантика кнопки «Продать»).
- `ErrorInPosition` в `sellFailReasons` добавляется производно: если нет нативных cart-level причин, но хотя бы у одной позиции `sellFailReasons` не пуст (см. [[#SellCardFailReasonEnum]]).
- При пустом `positions` в `sellFailReasons` включается `EmptySellCard`.

**Связано**
[[#SellCardPosition]], [[#SellCardFailReasonEnum]], [[#CreateSellCardResult]], [[#RecreateSellCardResult]], [[#GetActiveSellCardResult]]

### SellCardPosition

**Сигнатура**
```
class SellCardPosition
{
	string uid;
	string name;
	string prefabName;
	string[] entityUids;
	int quantity;
	int unitPrice;
	int lineSum;
	SellCardPositionFailReasonEnum[] sellFailReasons;
	string[] allowedShopUids;
}
```

**Назначение**
Одна агрегированная строка корзины продажи. В API-ответе содержит `uid`, `name`, `prefabName`, `entityUids`, `quantity`, `unitPrice`, `lineSum`, `sellFailReasons`, `allowedShopUids`.

**Свойства**
- `uid: string` - идентификатор строки в корзине продажи.
- `name: string` - отображаемое имя позиции (как у [[#ShopProduct]] `name`; обычно совпадает с именем товара; приходит в данных позиции вместе с агрегатом строки).
- `prefabName: string` - имя префаба для строки; приходит в данных позиции в составе актуального состояния корзины.
- `entityUids: string[]` - uid игровых сущностей в строке. Клиент получает это поле в `SellCardPosition`, но не передаёт `entityUids` в методы добавления или изменения количества.
- `quantity: int` - количество для отображения и проверок; приходит в агрегированных данных позиции/корзины.
- `unitPrice: int` - цена выкупа за единицу на момент формирования или обновления строки (в минимальных денежных единицах мира / условных единицах — по правилам проекта); согласована с ценой из соответствующего агрегата инвентаря.
- `lineSum: int` - сумма по строке с учётом `quantity` (`unitPrice * quantity`; те же единицы, что у [[#ShopProduct]] `unitPrice`); пересчитывается TradeManager при [[TradeManager.SellMethods.md#addToSellCard]] и [[TradeManager.SellMethods.md#changeSellCardPositionQuantity]].
- `sellFailReasons: SellCardPositionFailReasonEnum[]` - причины, по которым эту позицию сейчас нельзя продать (position-level: владение, доступность продажи, превышение инвентаря и т.п.). Cart-level причины (дубликаты `entityUid`, неположительная сумма корзины) сюда не входят — только в `sellFailReasons` у [[#SellCard]].
- `allowedShopUids: string[]` - при `CannotSellEntityInShop` в `sellFailReasons` — список магазинов, где продажа разрешена; иначе пустой / не используется.

**Связано**
[[#SellCard]], [[#SellCardPositionFailReasonEnum]]

### SellCardFailReasonEnum

**Сигнатура**
```
enum SellCardFailReasonEnum
{
	case CannotSellCardHasDuplicateEntityUid;
	case CannotSellCardPriceNotPositive;
	case EmptySellCard;
	case ErrorInPosition;
}
```

**Назначение**
Причины отказа в продаже на уровне корзины (`SellCard.sellFailReasons`). Отдельный enum от [[#SellCardPositionFailReasonEnum]].

**Значения**
- `CannotSellCardHasDuplicateEntityUid` - в корзине обнаружены повторяющиеся `entityUid` среди всех позиций и всех списков `entityUids`.
- `CannotSellCardPriceNotPositive` - итоговая сумма продажи недопустима (`totalSum` не положителен).
- `EmptySellCard` - в корзине нет позиций.
- `ErrorInPosition` - производная причина: нативных cart-level причин нет, но хотя бы у одной [[#SellCardPosition]] `sellFailReasons` не пуст. Детали — в `sellFailReasons` позиций.

**Используется в**
[[#SellCard]]

### SellCardPositionFailReasonEnum

**Сигнатура**
```
enum SellCardPositionFailReasonEnum
{
	case CannotSellEntity;
	case CannotSellEntityInShop;
	case CannotSellItemNotOwnedByCharacter;
	case QuantityExceedsInventory;
	case ContainerIsNotEmpty;
}
```

**Назначение**
Причины отказа в продаже на уровне одной позиции (`SellCardPosition.sellFailReasons`) и строки [[#InventoryProductPosition]] (`sellFailReasons`).

**Значения**
- `CannotSellEntity` - предмет нельзя продать ни в одном магазине.
- `CannotSellEntityInShop` - предмет нельзя продать в текущем магазине.
- `CannotSellItemNotOwnedByCharacter` - предмет не принадлежит персонажу.
- `QuantityExceedsInventory` - `quantity` позиции превышает доступное количество в инвентаре.
- `ContainerIsNotEmpty` - контейнер непустой: в нём есть вложенные предметы; такую строку нельзя продать, пока контейнер не опустеет.

**Используется в**
[[#SellCardPosition]], [[#InventoryProductPosition]], [[#ChangeSellCardPositionQuantityFailReasonEnum]]

### CreateSellCardResult

**Сигнатура**
```
class CreateSellCardResult
{
	OperationStatusEnum status;
	CreateSellCardFailReasonEnum failReason;
	SellCard card;
}
```

**Связано**
[[#OperationStatusEnum]], [[#CreateSellCardFailReasonEnum]], [[#SellCard]]

### CreateSellCardFailReasonEnum

**Сигнатура**
```
enum CreateSellCardFailReasonEnum
{
	case CannotCreateSellCardAlreadyExists;
}
```

**Назначение**
Причины отказа при создании корзины продажи.

**Значения**
- `CannotCreateSellCardAlreadyExists` - у персонажа уже есть корзина продажи.

**Используется в**
[[#CreateSellCardResult]]

### RecreateSellCardResult

**Сигнатура**
```
class RecreateSellCardResult
{
	OperationStatusEnum status;
	SellCard card;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#recreateSellCard]].

**Свойства**
- `status: OperationStatusEnum` - итог пересоздания корзины продажи.
- `card: SellCard` - при `status = Ok` содержит новую корзину; при `status = Fail` не используется.

**Связано**
[[#OperationStatusEnum]], [[#SellCard]]

### GetActiveSellCardResult

**Сигнатура**
```
class GetActiveSellCardResult
{
	OperationStatusEnum status;
	GetActiveSellCardFailReasonEnum failReason;
	SellCard card;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#getActiveSellCard]].

**Свойства**
- `status: OperationStatusEnum` - найдена ли активная корзина продажи.
- `failReason: GetActiveSellCardFailReasonEnum` - причина при `status = Fail`.
- `card: SellCard` - при `status = Ok` содержит активную корзину; при `status = Fail` не используется.

**Связано**
[[#OperationStatusEnum]], [[#GetActiveSellCardFailReasonEnum]], [[#SellCard]]

### GetActiveSellCardFailReasonEnum

**Сигнатура**
```
enum GetActiveSellCardFailReasonEnum
{
	case NoActiveSellCard;
}
```

**Назначение**
Причины отказа при запросе активной корзины продажи.

**Значения**
- `NoActiveSellCard` - у персонажа нет активной корзины продажи.

**Используется в**
[[#GetActiveSellCardResult]]

### AddToSellCardResult

**Сигнатура**
```
class AddToSellCardResult
{
	OperationStatusEnum status;
	AddToSellCardFailReasonEnum failReason;
	string positionUid;
	string[] allowedShopUids;
}
```

**Свойства**
- `status: OperationStatusEnum` - итог добавления в корзину продажи.
- `failReason: AddToSellCardFailReasonEnum` - причина отказа при `status = Fail`.
- `positionUid: string` - идентификатор строки [[#SellCardPosition]] (`uid`), к которой относится результат после обновления корзины (новая или обновлённая агрегированная строка).
- `allowedShopUids: string[]` - список магазинов, где продажа разрешена. Заполняется при `failReason = CannotSellEntityInShop`.

**Связано**
[[#OperationStatusEnum]], [[#AddToSellCardFailReasonEnum]]

### AddToSellCardFailReasonEnum

**Сигнатура**
```
enum AddToSellCardFailReasonEnum
{
	case CannotAddInventoryPositionNotFound;
	case CannotAddQuantityExceedsInventory;
	case CannotUseNonPositiveQuantity;
	case CannotSellEntity;
	case CannotSellEntityInShop;
	case ContainerIsNotEmpty;
}
```

**Назначение**
Причины отказа при добавлении в корзину продажи.

**Значения**
- `CannotAddInventoryPositionNotFound` - не найдена агрегированная строка инвентаря с указанным `inventoryPositionUid` в контексте персонажа и магазина корзины.
- `CannotAddQuantityExceedsInventory` - запрошенное количество вместе с уже учтёнными в [[#SellCard]] сущностями того же агрегата превышает `quantity` строки инвентаря.
- `CannotUseNonPositiveQuantity` - количество должно быть больше 0.
- `CannotSellEntity` - сущности агрегата нельзя продать ни в одном магазине.
- `CannotSellEntityInShop` - сущности агрегата нельзя продать в текущем магазине.
- `ContainerIsNotEmpty` - агрегат — непустые контейнеры; в корзину нельзя добавить контейнер, в котором есть вложенные предметы.

**Используется в**
[[#AddToSellCardResult]]

### ChangeSellCardPositionQuantityResult

**Сигнатура**
```
class ChangeSellCardPositionQuantityResult
{
	OperationStatusEnum status;
	ChangeSellCardPositionQuantityFailReasonEnum failReason;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#changeSellCardPositionQuantity]].

**Связано**
[[#OperationStatusEnum]], [[#ChangeSellCardPositionQuantityFailReasonEnum]]

### ChangeSellCardPositionQuantityFailReasonEnum

**Сигнатура**
```
enum ChangeSellCardPositionQuantityFailReasonEnum
{
	case CannotUseNonPositiveQuantity;
	case QuantityExceedsInventory;
	case CannotSellEntity;
	case CannotSellEntityInShop;
	case CannotSellItemNotOwnedByCharacter;
}
```

**Назначение**
Причины отказа при изменении количества позиции корзины продажи. Значения `QuantityExceedsInventory`, `CannotSellEntity`, `CannotSellEntityInShop`, `CannotSellItemNotOwnedByCharacter` согласованы по смыслу с [[#SellCardPositionFailReasonEnum]].

**Значения**
- `CannotUseNonPositiveQuantity` - новое количество должно быть больше 0.
- `QuantityExceedsInventory` - запрошенное `newQuantity` вместе с сущностями того же агрегата в других строках корзины превышает фактический `quantity` в инвентаре; `quantity` откатывается к максимально допустимому.
- `CannotSellEntity` - сущности позиции нельзя продать ни в одном магазине.
- `CannotSellEntityInShop` - сущности позиции нельзя продать в текущем магазине.
- `CannotSellItemNotOwnedByCharacter` - сущности позиции не принадлежат персонажу.

**Используется в**
[[#ChangeSellCardPositionQuantityResult]]

### SellResult

**Сигнатура**
```
class SellResult
{
	OperationStatusEnum status;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#sell]]. При `status = Fail` причины отказа не возвращаются — клиент перезапрашивает корзину через [[TradeManager.SellMethods.md#getActiveSellCard]].

**Свойства**
- `status: OperationStatusEnum` - итог операции продажи.

**Связано**
[[#OperationStatusEnum]]

### ShopCategory

**Сигнатура**
```
class ShopCategory
{
	string uid;
	string name;
}
```

**Назначение**
Категория товаров в каталоге магазина.

**Свойства**
- `uid: string` - идентификатор категории.
- `name: string` - отображаемое имя категории.

**Связано**
[[#GetShopCategoriesResult]]

### GetShopCategoriesResult

**Сигнатура**
```
class GetShopCategoriesResult
{
	OperationStatusEnum status;
	GetShopCategoriesFailReasonEnum failReason;
	ShopCategory[] categories;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.ShopMethods.md#getShopCategories]].

**Свойства**
- `status: OperationStatusEnum` - итог запроса категорий.
- `failReason: GetShopCategoriesFailReasonEnum` - причина при `status = Fail`.
- `categories: ShopCategory[]` - список категорий при `status = Ok`; при `Fail` не используется.

**Связано**
[[#OperationStatusEnum]], [[#GetShopCategoriesFailReasonEnum]], [[#ShopCategory]]

### GetShopCategoriesFailReasonEnum

**Сигнатура**
```
enum GetShopCategoriesFailReasonEnum
{
	case ShopNotFound;
}
```

**Назначение**
Причины отказа при запросе категорий магазина.

**Значения**
- `ShopNotFound` - магазин с указанным `shopUid` не найден или недоступен.

**Используется в**
[[#GetShopCategoriesResult]]

### ShopProduct

**Сигнатура**
```
class ShopProduct
{
	string uid;
	string prefabName;
	string name;
	int unitPrice;
	int availableQuantity;
	bool isAvailableForBuy;
}
```

**Назначение**
Позиция товара в каталоге магазина (в рамках категории и контекста персонажа).

**Свойства**
- `uid: string` - идентификатор предложения в каталоге; передаётся в [[TradeManager.BuyMethods.md#addToBuyCard]] как `shopProductUid`.
- `prefabName: string` - имя префаба товара.
- `name: string` - отображаемое имя.
- `unitPrice: int` - цена за единицу (в минимальных денежных единицах мира / условных единицах — по правилам проекта).
- `availableQuantity: int` - доступное количество к покупке (семантика — по правилам проекта: склад, лимит и т.п.).
- `isAvailableForBuy: bool` - можно ли добавить товар в корзину покупки в текущих условиях.

**Связано**
[[#ShopProductsResult]]

### ShopProductsResult

**Сигнатура**
```
class ShopProductsResult
{
	OperationStatusEnum status;
	ShopProductsFailReasonEnum failReason;
	ShopProduct[] products;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.ShopMethods.md#getProductsForBuyByCharacter]].

**Свойства**
- `status: OperationStatusEnum` - итог запроса товаров.
- `failReason: ShopProductsFailReasonEnum` - причина при `status = Fail`.
- `products: ShopProduct[]` - список товаров при `status = Ok`; при `Fail` не используется.

**Связано**
[[#OperationStatusEnum]], [[#ShopProductsFailReasonEnum]], [[#ShopProduct]]

### ShopProductsFailReasonEnum

**Сигнатура**
```
enum ShopProductsFailReasonEnum
{
	case ShopNotFound;
	case CategoryNotFound;
	case CharacterNotFound;
}
```

**Назначение**
Причины отказа при запросе товаров магазина.

**Значения**
- `ShopNotFound` - магазин с указанным `shopUid` не найден или недоступен.
- `CategoryNotFound` - категория с указанным `categoryUid` не найдена в этом магазине.
- `CharacterNotFound` - персонаж с указанным `characterUid` не найден или недоступен для расчёта каталога.

**Используется в**
[[#ShopProductsResult]]

### GetDescriptionByPrefabResult

**Сигнатура**
```
class GetDescriptionByPrefabResult
{
	OperationStatusEnum status;
	GetDescriptionByPrefabFailReasonEnum failReason;
	string description;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.ShopMethods.md#getDescriptionByPrefab]].

**Свойства**
- `status: OperationStatusEnum` - итог запроса описания.
- `failReason: GetDescriptionByPrefabFailReasonEnum` - причина при `status = Fail`.
- `description: string` - текстовое описание товара при `status = Ok`; при `Fail` не используется. Пустая строка допустима при `Ok`, если префаб известен каталогу, но описание не задано.

**Связано**
[[#OperationStatusEnum]], [[#GetDescriptionByPrefabFailReasonEnum]]

### GetDescriptionByPrefabFailReasonEnum

**Сигнатура**
```
enum GetDescriptionByPrefabFailReasonEnum
{
	case PrefabNotFound;
}
```

**Назначение**
Причины отказа при запросе текстового описания по префабу.

**Значения**
- `PrefabNotFound` - префаб с указанным `prefabName` не найден в каталоге TradeManager или для него нет метаданных описания.

**Используется в**
[[#GetDescriptionByPrefabResult]]

### GetInventoryForSellResult

**Сигнатура**
```
class GetInventoryForSellResult
{
	OperationStatusEnum status;
	GetInventoryForSellFailReasonEnum failReason;
	InventoryForSell inventoryForSell;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#getInventoryForSell]].

**Свойства**
- `status: OperationStatusEnum` - итог запроса снимка инвентаря для продажи.
- `failReason: GetInventoryForSellFailReasonEnum` - причина при `status = Fail` (магазин или персонаж недоступны).
- `inventoryForSell: InventoryForSell` - при `status = Ok` содержит агрегированные позиции и сумму; при `status = Fail` не используется (например пустой агрегат).

**Связано**
[[#OperationStatusEnum]], [[#GetInventoryForSellFailReasonEnum]], [[#InventoryForSell]]

### GetInventoryForSellFailReasonEnum

**Сигнатура**
```
enum GetInventoryForSellFailReasonEnum
{
	case ShopNotFound;
	case CharacterNotFound;
}
```

**Назначение**
Причины отказа при запросе инвентаря персонажа для продажи в контексте магазина.

**Значения**
- `ShopNotFound` - магазин с указанным `shopUid` не найден или недоступен (те же смысл и имя, что в [[#ShopProductsFailReasonEnum]]).
- `CharacterNotFound` - персонаж с указанным `characterUid` не найден или недоступен для расчёта (те же смысл и имя, что в [[#ShopProductsFailReasonEnum]]).

**Используется в**
[[#GetInventoryForSellResult]]

### InventoryForSell

**Сигнатура**
```
class InventoryForSell
{
	InventoryProductPosition[] positions;
	int totalAmount;
}
```

**Назначение**
Агрегированный снимок инвентаря персонажа для UI продажи в выбранном магазине: строки по префабу и общая сумма по тем строкам, которые попали в `positions` с учётом флага `isExcludeNotAvailableForSell` в [[TradeManager.SellMethods.md#getInventoryForSell]]. Поле `quantity` в строках отражает фактическое количество в инвентаре; до успешного `sell` оно не уменьшается из‑за позиций в корзине.

**Свойства**
- `positions: InventoryProductPosition[]` - агрегированные строки (после фильтрации по `isExcludeNotAvailableForSell`, если он задан в `true`).
- `totalAmount: int` - сумма `lineSum` по всем элементам `positions` (в минимальных денежных единицах мира / условных единицах — по правилам проекта); при пустом `positions` допускается `0`.

**Связано**
[[#InventoryProductPosition]], [[#GetInventoryForSellResult]]

### InventoryProductPosition

**Сигнатура**
```
class InventoryProductPosition
{
	string uid;
	string name;
	string prefabName;
	int quantity;
	int unitPrice;
	int lineSum;
	SellCardPositionFailReasonEnum[] sellFailReasons;
	string[] allowedShopUids;
}
```

**Назначение**
Одна агрегированная строка снимка инвентаря для продажи: идентификатор строки, префаб, фактическое количество в инвентаре, цена выкупа за единицу и сумма по строке в контексте `shopUid`, причины недоступности к продаже и список разрешённых магазинов. Симметрия с [[#ShopProduct]]: `uid` передаётся в [[TradeManager.SellMethods.md#addToSellCard]] как `inventoryPositionUid`, `unitPrice` — цена за единицу в контексте магазина.

**Свойства**
- `uid: string` - стабильный идентификатор агрегированной строки в контексте `(characterUid, shopUid, ключ агрегации)`; тот же id, что передаётся в [[TradeManager.SellMethods.md#addToSellCard]] как `inventoryPositionUid`. Стабилен между повторными вызовами [[TradeManager.SellMethods.md#getInventoryForSell]], пока не меняется состав агрегата. Ключ агрегации = `prefabName`; пустые и непустые контейнеры одного префаба — разные строки (у непустых ключ дополняется признаком непустого контейнера).
- `name: string` - отображаемое имя (как у [[#ShopProduct]]; источник — по правилам проекта). У строки непустых контейнеров в конце добавляется суффикс ` (не пустой)`.
- `prefabName: string` - имя префаба для строки.
- `quantity: int` - фактическое количество единиц в агрегате у персонажа в инвентаре. До успешного [[TradeManager.SellMethods.md#sell]] не уменьшается из‑за сущностей, уже учтённых в [[#SellCard]].
- `unitPrice: int` - цена выкупа за единицу в контексте магазина из [[TradeManager.SellMethods.md#getInventoryForSell]] (в минимальных денежных единицах мира / условных единицах — по правилам проекта; тот же смысл, что у [[#ShopProduct]] `unitPrice`).
- `lineSum: int` - сумма выкупа по строке с учётом `quantity` (`unitPrice * quantity`; те же единицы, что поле `lineSum` у [[#SellCardPosition]]).
- `sellFailReasons: SellCardPositionFailReasonEnum[]` - причины недоступности продажи; на уровне строки инвентаря используются `CannotSellEntity`, `CannotSellEntityInShop`, `ContainerIsNotEmpty`. Пустой массив означает, что строка доступна к продаже в текущем контексте. Пустые контейнеры и не-контейнеры одного `prefabName` — одна строка; непустые контейнеры того же префаба — отдельная строка с `ContainerIsNotEmpty`.
- `allowedShopUids: string[]` - при `CannotSellEntityInShop` в `sellFailReasons` — список магазинов, где продажа разрешена (как у [[#AddToSellCardResult]] / [[#SellCardPosition]]); иначе пустой массив или не используется клиентом.

**Связано**
[[#SellCardPosition]], [[#SellCardPositionFailReasonEnum]], [[#InventoryForSell]]

**Используется в**
[[#InventoryForSell]]