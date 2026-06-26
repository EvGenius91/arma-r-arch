## Сущности TradeManager

Назад: [[TradeManager.md]]

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
[[#AddToBuyCardResult]], [[#ChangeBuyCardPositionQuantityResult]], [[#BuyStatus]], [[#CanBuyStatus]], [[#CreateBuyCardResult]], [[#RecreateBuyCardResult]], [[#GetActiveBuyCardResult]], [[#CreateSellCardResult]], [[#RecreateSellCardResult]], [[#GetActiveSellCardResult]], [[#AddToSellCardResult]], [[#CanSellStatus]], [[#SellResult]], [[#GetShopCategoriesResult]], [[#ShopProductsResult]], [[#GetInventoryForSellResult]]

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
}
```

**Назначение**
Корзина покупки для одного персонажа. Одновременно у персонажа может существовать только одна корзина.

**Свойства**
- `uid: string` - уникальный идентификатор корзины.
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.
- `positions: BuyCardPosition[]` - позиции товаров в корзине.
- `deliveryMethod: DeliveryMethod` - выбранный способ доставки.
- `type: BuyCardTypeEnum` - тип корзины.

**Инварианты**
- На персонажа разрешена только одна активная корзина.
- `deliveryMethod` должен быть совместим с `type`.
- Среди позиций в `positions` значение `shopProductUid` не повторяется: для одного предложения каталога одна строка корзины, повторное добавление увеличивает её `quantity`.

**Связано**
[[#BuyCardPosition]], [[#DeliveryMethod]], [[#BuyCardTypeEnum]], [[#GetActiveBuyCardResult]]

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

**Инварианты**
- `quantity` должно быть больше 0.
- В пределах одной [[#BuyCard]] каждое `shopProductUid` встречается не более чем в одной позиции.
- При `quantity > 0` ожидается `lineSum >= 0`; при нулевой или отрицательной цене предложения — по правилам проекта (обычно такие товары не попадают в корзину через [[TradeManager.BuyMethods.md#addToBuyCard]]).

**Используется в**
[[#BuyCard]]

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
[[#BuyCard]], [[#CreateBuyCardFailReasonEnum]], [[#BuyFailReasonEnum]], [[#CanBuyFailReasonEnum]]

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
	case CannotAddIncompatibleBuyCardType;
	case CannotUseNonPositiveQuantity;
}
```

**Значения**
- `CannotAddShopProductNotFound` - в каталоге TradeManager нет предложения с указанным `shopProductUid` (неизвестный или снятый с продажи uid).
- `CannotAddShopProductNotInShop` - предложение есть, но не относится к магазину корзины (`BuyCard.shopUid`).
- `CannotAddShopProductNotAvailableForBuy` - предложение в текущем состоянии нельзя добавить в корзину (недоступно к покупке, нехватка остатка и т.п. — по правилам проекта).
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
}
```

**Назначение**
Причины отказа при изменении количества позиции корзины покупки.

**Значения**
- `CannotUseNonPositiveQuantity` - новое количество должно быть больше 0.

**Используется в**
[[#ChangeBuyCardPositionQuantityResult]]

### BuyStatus

**Сигнатура**
```
class BuyStatus
{
	OperationStatusEnum status;
	BuyFailReasonEnum failReason;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#buy]].

**Свойства**
- `status: OperationStatusEnum` - итог операции покупки.
- `failReason: BuyFailReasonEnum` - причина при `status = Fail`.

**Связано**
[[#OperationStatusEnum]], [[#BuyFailReasonEnum]]

### BuyFailReasonEnum

**Сигнатура**
```
enum BuyFailReasonEnum
{
	case CannotAfford;
	case CannotFitInInventory;
	case CannotSpawnVehicleNoFreeSlot;
	case CannotUseDeliveryMethodForBuyCardType;
}
```

**Назначение**
Причины отказа в покупке.

**Значения**
- `CannotAfford` - у персонажа недостаточно денег.
- `CannotFitInInventory` - у персонажа недостаточно места в инвентаре.
- `CannotSpawnVehicleNoFreeSlot` - нет свободного слота для спавна техники.
- `CannotUseDeliveryMethodForBuyCardType` - тип доставки не совместим с типом корзины.

**Используется в**
[[#BuyStatus]]

### CanBuyStatus

**Сигнатура**
```
class CanBuyStatus
{
	OperationStatusEnum status;
	CanBuyFailReasonEnum reason;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.BuyMethods.md#canBuyCard]].

**Связано**
[[#OperationStatusEnum]], [[#CanBuyFailReasonEnum]]

### CanBuyFailReasonEnum

**Сигнатура**
```
enum CanBuyFailReasonEnum
{
	case CannotAfford;
	case CannotFitInInventory;
	case CannotSpawnVehicleNoFreeSlot;
	case CannotUseDeliveryMethodForBuyCardType;
}
```

**Назначение**
Причины, по которым корзину сейчас нельзя купить.

**Значения**
- `CannotAfford` - у персонажа недостаточно денег.
- `CannotFitInInventory` - у персонажа недостаточно места в инвентаре.
- `CannotSpawnVehicleNoFreeSlot` - нет свободного слота для спавна техники.
- `CannotUseDeliveryMethodForBuyCardType` - тип доставки не совместим с типом корзины.

**Используется в**
[[#CanBuyStatus]]

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
}
```

**Назначение**
Корзина продажи для одного персонажа. Одновременно у персонажа может существовать только одна корзина продажи.

**Инварианты**
- На персонажа разрешена только одна активная корзина продажи.
- Каждый `entityUid` из массивов `entityUids` всех позиций встречается в корзине не более одного раза (без повторов между строками и внутри списков).

**Связано**
[[#SellCardPosition]], [[#CreateSellCardResult]], [[#RecreateSellCardResult]], [[#GetActiveSellCardResult]]

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
	int lineSum;
}
```

**Назначение**
Одна агрегированная строка корзины продажи для UI: имя, префаб, количество, список сущностей, сумма по строке.

**Свойства**
- `uid: string` - идентификатор строки в корзине продажи.
- `name: string` - отображаемое имя позиции (как у [[#ShopProduct]] `name`; обычно совпадает с именем товара; приходит в данных позиции вместе с агрегатом строки).
- `prefabName: string` - имя префаба для строки; приходит в данных позиции в составе актуального состояния корзины.
- `entityUids: string[]` - uid игровых сущностей в строке; состав и агрегация задаются в данных корзины (не передаётся вызывающим кодом в TradeManager при [[TradeManager.SellMethods.md#addToSellCard]]).
- `quantity: int` - количество для отображения и проверок; приходит в агрегированных данных позиции/корзины (не выводится на клиенте из длины `entityUids`).
- `lineSum: int` - сумма по строке для блока корзины (минимальная денежная единица мира / условные единицы — по правилам проекта); приходит в данных позиции вместе с агрегатом строки.

**Связано**
[[#SellCard]]

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
- `status: OperationStatusEnum` - итог добавления сущности в корзину продажи.
- `failReason: AddToSellCardFailReasonEnum` - причина отказа при `status = Fail`.
- `positionUid: string` - идентификатор строки [[#SellCardPosition]] (`uid`), к которой относится результат после обновления корзины (новая или обновлённая агрегированная строка).
- `allowedShopUids: string[]` - список магазинов, где сущность можно продать. Заполняется при `failReason = CannotSellEntityInShop`.

**Связано**
[[#OperationStatusEnum]], [[#AddToSellCardFailReasonEnum]]

### AddToSellCardFailReasonEnum

**Сигнатура**
```
enum AddToSellCardFailReasonEnum
{
	case CannotAddSellEntityNotFound;
	case CannotAddDuplicateEntityUid;
	case CannotSellEntity;
	case CannotSellEntityInShop;
}
```

**Назначение**
Причины отказа при добавлении сущности в корзину продажи.

**Значения**
- `CannotAddSellEntityNotFound` - не найдена сущность с указанным `entityUid`.
- `CannotAddDuplicateEntityUid` - сущность с таким `entityUid` уже учтена в корзине продажи (в любой строке, в любой из `entityUids`).
- `CannotSellEntity` - сущность нельзя продать ни в одном магазине.
- `CannotSellEntityInShop` - сущность нельзя продать в текущем магазине.

**Используется в**
[[#AddToSellCardResult]]

### CanSellStatus

**Сигнатура**
```
class CanSellStatus
{
	OperationStatusEnum status;
	SellFailReasonEnum reason;
	string[] allowedShopUids;
}
```

**Свойства**
- `status: OperationStatusEnum` - итог проверки возможности продажи.
- `reason: SellFailReasonEnum` - причина отказа при `status = Fail`.
- `allowedShopUids: string[]` - список магазинов, где продажа разрешена. Заполняется при `reason = CannotSellEntityInShop`.

**Связано**
[[#OperationStatusEnum]], [[#SellFailReasonEnum]]

### SellResult

**Сигнатура**
```
class SellResult
{
	OperationStatusEnum status;
	SellFailReasonEnum failReason;
	string[] allowedShopUids;
}
```

**Назначение**
Результат выполнения метода [[TradeManager.SellMethods.md#sell]].

**Свойства**
- `status: OperationStatusEnum` - итог операции продажи.
- `failReason: SellFailReasonEnum` - причина отказа при `status = Fail`.
- `allowedShopUids: string[]` - список магазинов, где продажа разрешена. Заполняется при `failReason = CannotSellEntityInShop`.

**Связано**
[[#OperationStatusEnum]], [[#SellFailReasonEnum]]

### SellFailReasonEnum

**Сигнатура**
```
enum SellFailReasonEnum
{
	case CannotSellEntity;
	case CannotSellEntityInShop;
	case CannotSellItemNotOwnedByCharacter;
	case CannotSellCardHasDuplicateEntityUid;
	case CannotSellCardPriceNotPositive;
}
```

**Назначение**
Общий тип причин отказа при проверке корзины продажи ([[TradeManager.SellMethods.md#canSellCard]], поле `reason` у [[#CanSellStatus]]) и при выполнении продажи ([[TradeManager.SellMethods.md#sell]], поле `failReason` у [[#SellResult]]); те же значения допускаются на уровне строки [[#InventoryProductPosition]] при `isAvailableForSell = false` (см. описание поля `sellFailReason`).

**Значения**
- `CannotSellEntity` - один или несколько объектов нельзя продать ни в одном магазине.
- `CannotSellEntityInShop` - один или несколько объектов нельзя продать в текущем магазине.
- `CannotSellItemNotOwnedByCharacter` - один или несколько объектов не принадлежат персонажу.
- `CannotSellCardHasDuplicateEntityUid` - в корзине обнаружены повторяющиеся `entityUid` среди всех позиций и всех списков `entityUids`.
- `CannotSellCardPriceNotPositive` - итоговая сумма продажи недопустима (сумма `lineSum` по позициям не положительна).

**Используется в**
[[#CanSellStatus]], [[#SellResult]], [[#InventoryProductPosition]]

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
Агрегированный снимок инвентаря персонажа для UI продажи в выбранном магазине: строки по префабу и общая сумма по тем строкам, которые попали в `positions` с учётом флага `isExcludeNotAvailableForSell` в [[TradeManager.SellMethods.md#getInventoryForSell]].

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
	string name;
	string prefabName;
	string[] entityUids;
	int quantity;
	int lineSum;
	bool isAvailableForSell;
	SellFailReasonEnum sellFailReason;
	string[] allowedShopUids;
}
```

**Назначение**
Одна агрегированная строка снимка инвентаря для продажи: префаб, количество, список сущностей, сумма по строке в контексте `shopUid`, признак доступности к продаже в этом магазине и детали отказа (согласованы с проверками [[TradeManager.SellMethods.md#canSellCard]] по смыслу `CannotSellEntity` / `CannotSellEntityInShop`).

**Свойства**
- `name: string` - отображаемое имя (как у [[#ShopProduct]]; источник — по правилам проекта).
- `prefabName: string` - имя префаба для строки.
- `entityUids: string[]` - uid игровых сущностей, вошедших в агрегат (для добавления в [[#SellCard]] через [[TradeManager.SellMethods.md#addToSellCard]]).
- `quantity: int` - количество в строке; согласовано с агрегатом и проверками (не обязательно равно длине `entityUids`, если семантика количества задаётся правилами проекта).
- `lineSum: int` - сумма выкупа по строке в контексте указанного магазина (те же единицы и смысл, что поле `lineSum` у [[#SellCardPosition]]).
- `isAvailableForSell: bool` - можно ли продать все сущности строки в текущем `shopUid` (и при необходимости «вообще» — по правилам проекта).
- `sellFailReason: SellFailReasonEnum` - при `isAvailableForSell = false` указывает причину недоступности; ожидаемые значения на уровне строки: `CannotSellEntity`, `CannotSellEntityInShop` (остальные значения [[#SellFailReasonEnum]] относятся к корзине целиком и на строке не используются). При `isAvailableForSell = true` не используется.
- `allowedShopUids: string[]` - при `isAvailableForSell = false` и `sellFailReason = CannotSellEntityInShop` — список магазинов, где продажа разрешена (как у [[#AddToSellCardResult]] / [[#CanSellStatus]]); иначе не используется.

**Связано**
[[#SellCardPosition]], [[#SellFailReasonEnum]], [[#InventoryForSell]]

**Используется в**
[[#InventoryForSell]]