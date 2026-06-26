## Методы продажи TradeManager

Назад: [[TradeManager.md]]
Сущности: [[TradeManager.Entities.md]]

### createSellCard

**Сигнатура**
createSellCard(string shopUid, string characterUid): [[TradeManager.Entities.md#CreateSellCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.

**Описание**
Создаёт [[TradeManager.Entities.md#SellCard]] для персонажа в указанном магазине: в ней затем набираются позиции на продажу. После появления строк они приходят в виде агрегированных [[TradeManager.Entities.md#SellCardPosition]] (`name`, `prefabName`, `quantity`, `entityUids`, `lineSum`) в данных корзины.

**Проверки**
- У персонажа не должно быть другой активной корзины продажи.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CannotCreateSellCardAlreadyExists`

### recreateSellCard

**Сигнатура**
recreateSellCard(string shopUid, string characterUid): [[TradeManager.Entities.md#RecreateSellCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.

**Описание**
Создаёт [[TradeManager.Entities.md#SellCard]] для персонажа в указанном магазине. Если у персонажа уже есть активная корзина продажи, она удаляется вместе со всеми позициями; возвращается **новая** корзина с новым `uid` и пустым `positions`. Клиент обязан использовать `card.uid` из ответа для последующих вызовов (`addToSellCard`, `canSellCard`, `sell` и т.д.); прежний `sellCardUid` после успешного вызова недействителен.

Для первого создания корзины без сброса существующей используйте [[#createSellCard]].

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.

### getActiveSellCard

**Сигнатура**
getActiveSellCard(string characterUid): [[TradeManager.Entities.md#GetActiveSellCardResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.

**Описание**
Возвращает текущую активную [[TradeManager.Entities.md#SellCard]] персонажа, если она есть. В `card.positions` — агрегированные строки ([[TradeManager.Entities.md#SellCardPosition]]: `name`, `prefabName`, `quantity`, `entityUids`, `lineSum`) для UI.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- Если активной корзины продажи нет: `status = Fail` и `failReason = NoActiveSellCard`.

**Ошибки / причины отказа**
- `NoActiveSellCard`

### getInventoryForSell

**Сигнатура**
getInventoryForSell(string characterUid, string shopUid, bool isExcludeNotAvailableForSell): [[TradeManager.Entities.md#GetInventoryForSellResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.
- `shopUid: string` - идентификатор магазина (контекст цен и правил продажи в этом магазине).
- `isExcludeNotAvailableForSell: bool` - если `true`, в ответ попадают только агрегированные строки, доступные для продажи в указанном `shopUid`; если `false` — все агрегированные строки инвентаря персонажа, с отметкой доступности и причиной / `allowedShopUids` на уровне строки (см. [[TradeManager.Entities.md#InventoryProductPosition]]).

**Описание**
Собирает элементы инвентаря персонажа (владение персонажем, объём инвентаря/контейнеров — по правилам проекта) и возвращает [[TradeManager.Entities.md#InventoryForSell]]: массив [[TradeManager.Entities.md#InventoryProductPosition]] с полями в духе [[TradeManager.Entities.md#SellCardPosition]] (`name`, `prefabName`, `quantity`, `entityUids`, `lineSum`) плюс признак и детали доступности к продаже в текущем магазине, а также [[TradeManager.Entities.md#InventoryForSell]] `totalAmount` — сумма `lineSum` по всем строкам в `positions` (после фильтра при `isExcludeNotAvailableForSell = true`).

**Результат**
- При успехе возвращает `status = Ok` и заполненный `inventoryForSell` (в т.ч. `positions` и `totalAmount`).
- При отказе возвращает `status = Fail`, `failReason` из [[TradeManager.Entities.md#GetInventoryForSellFailReasonEnum]]; `inventoryForSell` не используется.

**Ошибки / причины отказа**
- `ShopNotFound`
- `CharacterNotFound`

### addToSellCard

**Сигнатура**
addToSellCard(string sellCardUid, string entityUid): [[TradeManager.Entities.md#AddToSellCardResult]]

**Аргументы**
- `sellCardUid: string` - идентификатор корзины продажи.
- `entityUid: string` - уникальный идентификатор игровой сущности для продажи.

**Описание**
Префаб не передаётся параметром: задаются `sellCardUid` и `entityUid`. После успеха актуальное состояние корзины (включая агрегированные [[TradeManager.Entities.md#SellCardPosition]] с `name`, `prefabName`, `quantity`, `entityUids`, `lineSum`) приходит в данных [[TradeManager.Entities.md#SellCard]]; TradeManager на стороне игры не собирает строки и не подставляет префаб по локальной сущности. Владение предметами персонажем и положительная сумма продажи проверяются в [[#canSellCard]] и [[#sell]], а не в этом методе.

**Проверки**
- Сущность с `entityUid` должна существовать (иначе `CannotAddSellEntityNotFound`).
- `entityUid` не должен встречаться ни в одном списке `entityUids` ни одной позиции корзины (иначе `CannotAddDuplicateEntityUid`).
- Предмет должен быть доступен для продажи.
- Продажа должна быть разрешена в текущем магазине: у [[TradeManager.Entities.md#SellCard]] с идентификатором `sellCardUid` поле `shopUid` должно входить в `allowedShopUids` для добавляемой сущности.

**Ошибки / причины отказа**
- `CannotAddSellEntityNotFound`
- `CannotAddDuplicateEntityUid`
- `CannotSellEntity`
- `CannotSellEntityInShop`

**Результат**
- При успехе возвращает `status = Ok` и `positionUid` (строка [[TradeManager.Entities.md#SellCardPosition]], созданная или обновлённая после учёта сущности).
- При отказе с `CannotSellEntityInShop` возвращает `allowedShopUids` со списком магазинов, где продажа разрешена.

### removeSellCardPosition

**Сигнатура**
removeSellCardPosition(string sellCardPositionUid): void

**Аргументы**
- `sellCardPositionUid: string` - идентификатор удаляемой строки корзины (`SellCardPosition.uid`).

**Описание**
Удаляет из корзины **целую агрегированную строку** по её `uid`; частичное снятие одного `entityUid` из строки этим методом не выполняется.

### canSellCard

**Сигнатура**
canSellCard(string sellCardUid): [[TradeManager.Entities.md#CanSellStatus]]

**Аргументы**
- `sellCardUid: string` - идентификатор корзины продажи.

**Проверки**
- Для каждого `entityUid` из объединения всех массивов `entityUids` по позициям: сущность существует и в текущем состоянии допускает продажу (ограничения экземпляра: состояние объекта, блокировки и т.п. по правилам игры).
- Для каждой позиции: тип (`prefabName`) не запрещён к продаже глобально (правила мира / ассортимента).
- Каждый предмет должен быть разрешён к продаже в текущем магазине.
- Каждый предмет должен принадлежать персонажу.
- Среди всех позиций и всех списков `entityUids` не должно быть повторяющихся `entityUid`.
- Сумма `lineSum` по всем позициям должна быть больше 0.

**Ошибки / причины отказа**
- `CannotSellEntity`
- `CannotSellEntityInShop`
- `CannotSellItemNotOwnedByCharacter`
- `CannotSellCardHasDuplicateEntityUid`
- `CannotSellCardPriceNotPositive`

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе с `CannotSellEntityInShop` возвращает `allowedShopUids` со списком магазинов, где продажа разрешена.

### removeSellCard

**Сигнатура**
removeSellCard(string sellCardUid): void

**Аргументы**
- `sellCardUid: string` - идентификатор корзины продажи.

### sell

**Сигнатура**
sell(string sellCardUid): [[TradeManager.Entities.md#SellResult]]

**Аргументы**
- `sellCardUid: string` - идентификатор корзины продажи.

**Описание**
Продаёт все позиции из корзины продажи одним вызовом.

Поток применения в игровом мире после успешного ответа: [[BackendGameMutation.md]].

**Проверки**
- Все проверки из [[#canSellCard]] должны быть пройдены.

**Ошибки / причины отказа**
- `CannotSellEntity`
- `CannotSellEntityInShop`
- `CannotSellItemNotOwnedByCharacter`
- `CannotSellCardHasDuplicateEntityUid`
- `CannotSellCardPriceNotPositive`

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе с `CannotSellEntityInShop` возвращает `allowedShopUids` со списком магазинов, где продажа разрешена.
