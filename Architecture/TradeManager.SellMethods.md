## Методы продажи TradeManager

Назад: [[TradeManager.md]]
Сущности: [[TradeManager.Entities.md]]

Документ описывает публичное поведение JSON-RPC API.

### Поток UI продажи

Симметрия с покупкой: клиент передаёт `uid` агрегированной строки и `quantity`; конкретные `entityUids` приходят в ответе корзины, но не выбираются клиентом вручную.

1. [[#getInventoryForSell]] → список строк с `uid`, `quantity`
2. [[#getActiveSellCard]] → корзина с `sellFailReasons`
3. Добавление: `addToSellCard(sellCardUid, inventoryPositionUid, quantity)`
4. Изменение в корзине: `changeSellCardPositionQuantity(sellCardPositionUid, newQuantity)`
5. Переключение buy ↔ sell: перезапросить `getActiveSellCard` и `getInventoryForSell` — синхронизация `entityUid` на клиенте не нужна

### Пересчёт состояния корзины

TradeManager возвращает актуальный снимок корзины в поле `card` публичных API-ответов. В этом снимке пересчитаны `sellFailReasons`, `totalSum` у [[TradeManager.Entities.md#SellCard]] и `sellFailReasons` (и при необходимости `allowedShopUids`) у каждой [[TradeManager.Entities.md#SellCardPosition]].

Снимок возвращается в ответах:

- [[#getActiveSellCard]]
- [[#createSellCard]]
- [[#recreateSellCard]]

После мутаций, которые не возвращают `card`, клиент получает актуальное состояние через повторный вызов [[#getActiveSellCard]].

**Cart-level проверки** (в `SellCard.sellFailReasons`):
- `EmptySellCard` — если `positions` пуст;
- `CannotSellCardHasDuplicateEntityUid` — повторяющиеся `entityUid` среди всех позиций;
- `CannotSellCardPriceNotPositive` — `totalSum` не положителен.

**Position-level проверки** (в `SellCardPosition.sellFailReasons`):
- владение предметами (`CannotSellItemNotOwnedByCharacter`);
- доступность продажи (`CannotSellEntity`, `CannotSellEntityInShop`);
- превышение инвентаря (`QuantityExceedsInventory`);
- непустой контейнер (`ContainerIsNotEmpty`).

**Агрегация `ErrorInPosition`:** если нативных cart-level причин нет, но хотя бы у одной позиции `sellFailReasons` не пуст — в `SellCard.sellFailReasons` добавляется `ErrorInPosition`.

**Инвариант клиента:** `canSell ⟺ card.sellFailReasons` пуст. Пустой `sellFailReasons` не гарантирует успех [[#sell]] (возможна гонка); после `sell = Fail` клиент обязан вызвать [[#getActiveSellCard]].

**Refetch корзины:** клиент перезапрашивает [[#getActiveSellCard]] после успешного [[#addToSellCard]], после [[#changeSellCardPositionQuantity]], после удаления позиции или корзины, после `sell = Fail`, при изменении инвентаря персонажа, переключении buy/sell и по событиям с сервера.

### createSellCard

**Сигнатура**
createSellCard(string shopUid, string characterUid): [[TradeManager.Entities.md#CreateSellCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.

**Описание**
Создаёт [[TradeManager.Entities.md#SellCard]] для персонажа в указанном магазине. В `card` пересчитаны `sellFailReasons` (для пустой корзины — `EmptySellCard`), `totalSum = 0`.

**Проверки**
- У персонажа не должно быть другой активной корзины продажи.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
- `CannotCreateSellCardAlreadyExists`

### recreateSellCard

**Сигнатура**
recreateSellCard(string shopUid, string characterUid): [[TradeManager.Entities.md#RecreateSellCardResult]]

**Аргументы**
- `shopUid: string` - идентификатор магазина.
- `characterUid: string` - идентификатор персонажа.

**Описание**
Создаёт [[TradeManager.Entities.md#SellCard]] для персонажа в указанном магазине. Если у персонажа уже есть активная корзина продажи, она удаляется вместе со всеми позициями; возвращается **новая** корзина с новым `uid` и пустым `positions`. Клиент обязан использовать `card.uid` из ответа для последующих вызовов (`addToSellCard`, `sell` и т.д.); прежний `sellCardUid` после успешного вызова недействителен. В `card` пересчитаны `sellFailReasons` (для пустой корзины — `EmptySellCard`), `totalSum = 0`.

Для первого создания корзины без сброса существующей используйте [[#createSellCard]].

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.

### getActiveSellCard

**Сигнатура**
getActiveSellCard(string characterUid): [[TradeManager.Entities.md#GetActiveSellCardResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.

**Описание**
Возвращает текущую активную [[TradeManager.Entities.md#SellCard]] персонажа с пересчитанными `sellFailReasons`, `totalSum`. В `card.positions` — агрегированные строки ([[TradeManager.Entities.md#SellCardPosition]]: `uid`, `name`, `prefabName`, `entityUids`, `quantity`, `unitPrice`, `lineSum`, `sellFailReasons`, `allowedShopUids`) для UI.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `card`.
- Если активной корзины продажи нет: `status = Fail` и `failReason = NoActiveSellCard`.

**Причины отказа (`failReason`)**
- `NoActiveSellCard`

### getInventoryForSell

**Сигнатура**
getInventoryForSell(string characterUid, string shopUid, bool isExcludeNotAvailableForSell): [[TradeManager.Entities.md#GetInventoryForSellResult]]

**Аргументы**
- `characterUid: string` - идентификатор персонажа.
- `shopUid: string` - идентификатор магазина (контекст цен и правил продажи в этом магазине).
- `isExcludeNotAvailableForSell: bool` - если `true`, в ответ попадают только агрегированные строки, доступные для продажи в указанном `shopUid`; если `false` — все агрегированные строки инвентаря персонажа, с `sellFailReasons` / `allowedShopUids` на уровне строки (см. [[TradeManager.Entities.md#InventoryProductPosition]]).

**Описание**
Возвращает [[TradeManager.Entities.md#InventoryForSell]]: массив [[TradeManager.Entities.md#InventoryProductPosition]] с полями `uid`, `name`, `prefabName`, `quantity`, `unitPrice`, `lineSum`, `sellFailReasons`, `allowedShopUids`, а также `totalAmount` — сумму `lineSum` по всем строкам в `positions` (после фильтра при `isExcludeNotAvailableForSell = true`).

`quantity` в каждой строке — фактическое количество в инвентаре; до успешного [[#sell]] не уменьшается из‑за позиций в корзине. Повторный вызов после переключения режима покупки/продажи не требует клиентской синхронизации `entityUid`.

Пустые и непустые контейнеры одного `prefabName` — разные строки: у непустых в `sellFailReasons` есть `ContainerIsNotEmpty`, в `name` суффикс ` (не пустой)`.

**Результат**
- При успехе возвращает `status = Ok` и заполненный `inventoryForSell` (в т.ч. `positions` и `totalAmount`).
- При отказе возвращает `status = Fail`, `failReason` из [[TradeManager.Entities.md#GetInventoryForSellFailReasonEnum]]; `inventoryForSell` не используется.

**Причины отказа (`failReason`)**
- `ShopNotFound`
- `CharacterNotFound`

### addToSellCard

**Сигнатура**
addToSellCard(string sellCardUid, string inventoryPositionUid, int quantity): [[TradeManager.Entities.md#AddToSellCardResult]]

**Аргументы**
- `sellCardUid: string` - идентификатор корзины продажи.
- `inventoryPositionUid: string` - идентификатор агрегированной строки инвентаря (`uid` из [[TradeManager.Entities.md#InventoryProductPosition]], полученный из [[#getInventoryForSell]]).
- `quantity: int` - добавляемое количество.

**Описание**
TradeManager по `inventoryPositionUid` добавляет или обновляет [[TradeManager.Entities.md#SellCardPosition]] и возвращает `positionUid`. Симметрия с [[TradeManager.BuyMethods.md#addToBuyCard]]: клиент передаёт id строки каталога/инвентаря и количество.

После успеха клиент обязан вызвать [[#getActiveSellCard]] для актуальных `sellFailReasons`.

При превышении доступного количества в инвентаре операция отклоняется целиком — частичное добавление не выполняется.

**Проверки**
- `quantity` должно быть больше 0.
- Агрегированная строка с `inventoryPositionUid` должна существовать в контексте персонажа и магазина корзины (иначе `CannotAddInventoryPositionNotFound`).
- Суммарное количество добавляемых и уже учтённых в [[TradeManager.Entities.md#SellCard]] сущностей того же агрегата не должно превышать `quantity` строки инвентаря (иначе `CannotAddQuantityExceedsInventory`).
- Предметы агрегата должны быть доступны для продажи.
- Продажа должна быть разрешена в текущем магазине: у [[TradeManager.Entities.md#SellCard]] с идентификатором `sellCardUid` поле `shopUid` должно входить в `allowedShopUids` для добавляемого агрегата.
- Нельзя добавить непустой контейнер (`ContainerIsNotEmpty`).

**Причины отказа (`failReason`)**
- `CannotAddInventoryPositionNotFound`
- `CannotAddQuantityExceedsInventory`
- `CannotUseNonPositiveQuantity`
- `CannotSellEntity`
- `CannotSellEntityInShop`
- `ContainerIsNotEmpty`

**Результат**
- При успехе возвращает `status = Ok` и `positionUid` (строка [[TradeManager.Entities.md#SellCardPosition]], созданная или обновлённая после учёта сущностей).
- При отказе с `CannotSellEntityInShop` возвращает `allowedShopUids` со списком магазинов, где продажа разрешена.

### changeSellCardPositionQuantity

**Сигнатура**
changeSellCardPositionQuantity(string sellCardPositionUid, int newQuantity): [[TradeManager.Entities.md#ChangeSellCardPositionQuantityResult]]

**Аргументы**
- `sellCardPositionUid: string` - идентификатор позиции в корзине продажи.
- `newQuantity: int` - новое количество для позиции.

**Описание**
Меняет `quantity` строки корзины продажи. Ответ содержит только `status` и, при отказе, `failReason`; актуальный снимок корзины клиент получает через [[#getActiveSellCard]].

Если запрошенное `newQuantity` превышает доступное количество в инвентаре, `quantity` откатывается к максимально допустимому значению (не меньше 1), возвращается `status = Fail` и `failReason`. Актуальное `quantity` и причины клиент получает через [[#getActiveSellCard]] после ответа метода.

Полное удаление строки — через [[#removeSellCardPosition]], не через `newQuantity = 0`.

**Проверки**
- `newQuantity` должно быть больше 0.
- При увеличении: суммарное `newQuantity` по всем строкам корзины для того же агрегата не должно превышать `quantity` в инвентаре.
- Владение предметами, доступность продажи (как у position-level `sellFailReasons`).

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail` и `failReason`.

**Причины отказа (`failReason`)**
- `CannotUseNonPositiveQuantity`
- `QuantityExceedsInventory`
- `CannotSellEntity`
- `CannotSellEntityInShop`
- `CannotSellItemNotOwnedByCharacter`

### removeSellCardPosition

**Сигнатура**
removeSellCardPosition(string sellCardPositionUid): void

**Аргументы**
- `sellCardPositionUid: string` - идентификатор удаляемой строки корзины (`SellCardPosition.uid`).

**Описание**
Удаляет из корзины **целую агрегированную строку** по её `uid`. Частичное изменение количества — через [[#changeSellCardPositionQuantity]]. После вызова клиент перезапрашивает корзину через [[#getActiveSellCard]] для обновления `sellFailReasons`.

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
Продаёт все позиции из корзины продажи одним вызовом. Возвращает только `status`; при `Fail` причину не указывает. Клиент вызывает [[#getActiveSellCard]] для получения актуальных `sellFailReasons` и `sellFailReasons` позиций.

Допустимо, что `sell` вернёт `Fail`, даже если предыдущий снимок корзины показывал пустой `sellFailReasons` (гонка состояния).

**Проверки**
- Для каждого `entityUid` из объединения всех массивов `entityUids` по позициям: сущность существует и допускает продажу.
- Для каждой позиции: тип (`prefabName`) не запрещён к продаже глобально.
- Каждый предмет разрешён к продаже в текущем магазине.
- Каждый предмет принадлежит персонажу.
- Среди всех позиций и всех списков `entityUids` нет повторяющихся `entityUid`.
- Сумма `lineSum` по всем позициям (`totalSum`) больше 0.
- Position-level: превышение инвентаря и прочие проверки по позициям.

**Результат**
- При успехе возвращает `status = Ok`.
- При отказе возвращает `status = Fail`.
