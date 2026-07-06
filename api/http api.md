# HTTP API

Используется JSON-RPC 2.0: все запросы — `POST`, тело — JSON. Аутентификация по фиксированному ключу в HTTP-заголовке `X-Api-Key` (значение ключа выдаётся отдельно).

Рекомендуется заголовок `Content-Type: application/json`.

## Обёртка JSON-RPC 2.0

| Поле      | Значение                                              |
| --------- | ----------------------------------------------------- |
| `jsonrpc` | `"2.0"`                                               |
| `method`  | имя метода (строка)                                   |
| `params`  | объект с аргументами метода                           |
| `id`      | идентификатор запроса (число или строка)              |

Успешный ответ транспорта: HTTP 200, тело с полями `jsonrpc`, `result`, `id`. Итог операции TradeManager (`Ok` / `Fail`) передаётся внутри `result` в поле `status`, а не через объект JSON-RPC `error`.

## Методы

### getShopCategories

Возвращает список категорий товаров, доступных в указанном магазине. Подробнее: [Architecture/TradeManager.ShopMethods.md](../Architecture/TradeManager.ShopMethods.md#getshopcategories), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"getShopCategories"`

**`params`**

| Поле      | Тип    | Описание               |
| --------- | ------ | ---------------------- |
| `shopUid` | string | Идентификатор магазина |

**`result` (`GetShopCategoriesResult`)**

| Поле         | Тип            | Описание                                                          |
| ------------ | -------------- | ----------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                       |
| `failReason` | string \| null | При `Fail`: `"ShopNotFound"`; при `Ok` — `null`                   |
| `categories` | array \| null  | При `Ok`: массив `ShopCategory`; при `Fail` — `null`              |

Элемент `categories`: объекты с полями `uid`, `name`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "trade@getShopCategories",
  "params": {
    "shopUid": "shop-001"
  },
  "id": 1
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "categories": [
      {
        "uid": "cat-weapons",
        "name": "Оружие"
      },
      {
        "uid": "cat-ammo",
        "name": "Боеприпасы"
      }
    ]
  },
  "id": 1
}
```

**Пример ответа — отказ (магазин не найден)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "ShopNotFound",
    "categories": null
  },
  "id": 1
}
```

### getProductsForBuyByCharacter

Возвращает список товаров магазина в указанной категории с учётом контекста персонажа. Поле `uid` у каждого элемента `products` передаётся в [addToBuyCard](#addtobuycard) как `shopProductUid`. Подробнее: [Architecture/TradeManager.ShopMethods.md](../Architecture/TradeManager.ShopMethods.md#getproductsforbuybycharacter), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md) (`ShopProduct`, `ShopProductsResult`).

**`method`:** `"getProductsForBuyByCharacter"`

**`params`**

| Поле           | Тип    | Описание                                              |
| -------------- | ------ | ----------------------------------------------------- |
| `shopUid`      | string | Идентификатор магазина                                |
| `categoryUid`  | string | Идентификатор категории (`ShopCategory.uid`)          |
| `characterUid` | string | Идентификатор персонажа (контекст цен и доступности)  |

**`result` (`GetProductsForBuyByCharacterResult` / `ShopProductsResult`)**

| Поле         | Тип            | Описание                                                                                  |
| ------------ | -------------- | ----------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                               |
| `failReason` | string \| null | При `Fail`: `"ShopNotFound"`, `"CategoryNotFound"`, `"CharacterNotFound"`; при `Ok` — `null` |
| `products`   | array \| null  | При `Ok`: массив `ShopProduct`; при `Fail` — `null`                                       |

Элемент `products`: объекты с полями `uid`, `prefabName`, `name`, `unitPrice`, `availableQuantity`, `isAvailableForBuy` (числа: `unitPrice`, `availableQuantity`; `isAvailableForBuy` — bool).

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "trade@getProductsForBuyByCharacter",
  "params": {
    "shopUid": "shop-001",
    "categoryUid": "cat-weapons",
    "characterUid": "char-42"
  },
  "id": 1
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "products": [
      {
        "uid": "prod-ak74",
        "prefabName": "AK74",
        "name": "AK-74",
        "unitPrice": 5000,
        "availableQuantity": 10,
        "isAvailableForBuy": true
      },
      {
        "uid": "prod-m4",
        "prefabName": "M4A1",
        "name": "M4A1",
        "unitPrice": 6000,
        "availableQuantity": 5,
        "isAvailableForBuy": true
      }
    ]
  },
  "id": 1
}
```

**Пример ответа — отказ (категория не найдена)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CategoryNotFound",
    "products": null
  },
  "id": 1
}
```

### createBuyCard

Создаёт корзину покупки для персонажа в указанном магазине. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#createbuycard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"createBuyCard"`

**`params`**

| Поле             | Тип    | Описание                                                       |
| ---------------- | ------ | -------------------------------------------------------------- |
| `shopUid`        | string | Идентификатор магазина                                         |
| `characterUid`   | string | Идентификатор персонажа                                        |
| `type`           | string | `BuyCardTypeEnum`: `"Vehicle"` или `"Inventory"`               |
| `deliveryMethod` | string | `DeliveryMethod`: `"Inventory"` или `"NearVehicleSpawnPosition"` |

Совместимость: для `type: "Inventory"` допустим только `deliveryMethod: "Inventory"`; для `type: "Vehicle"` — только `"NearVehicleSpawnPosition"`.

Проверки: у персонажа не должно быть другой активной корзины покупки; тип корзины и способ доставки должны быть совместимы.

**`result` (`CreateBuyCardResult`)**

| Поле         | Тип            | Описание                                                                                                                                      |
| ------------ | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                                                                   |
| `failReason` | string \| null | При `Fail`: `"CannotCreateBuyCardAlreadyExists"` или `"CannotUseDeliveryMethodForBuyCardType"`; при `Ok` — `null`                             |
| `card`       | object \| null | При `Ok`: объект `BuyCard`; при `Fail` — `null`                                                                                               |

**`BuyCard` (в `result.card`)**

| Поле             | Тип    | Описание                                              |
| ---------------- | ------ | ----------------------------------------------------- |
| `uid`            | string | Идентификатор корзины                                 |
| `shopUid`        | string | Идентификатор магазина                                |
| `characterUid`   | string | Идентификатор персонажа                               |
| `positions`      | array  | Позиции (`BuyCardPosition`); при создании — `[]`      |
| `deliveryMethod` | string | Способ доставки                                       |
| `type`           | string | Тип корзины                                           |
| `buyFailReasons` | array  | Причины отказа на уровне корзины (`BuyCardFailReasonEnum`); при пустой корзине — `["EmptyBuyCard"]` |
| `moneyShortfall` | number | Недостающая сумма при `CannotAfford`; иначе `0`       |
| `totalSum`       | number | Сумма `lineSum` по всем позициям; при пустой корзине — `0` |

Элемент `positions`: объекты с полями `uid`, `shopProductUid`, `name`, `prefabName`, `quantity`, `lineSum`, `buyFailReasons` (числа: `quantity`, `lineSum`; `buyFailReasons` — массив строк `BuyCardPositionFailReasonEnum`).

Пример элемента `positions` (непустая корзина):

```json
{
  "uid": "buy-pos-7c2a",
  "shopProductUid": "prod-ak74",
  "name": "AK-74",
  "prefabName": "AK74",
  "quantity": 3,
  "lineSum": 15000,
  "buyFailReasons": []
}
```

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "createBuyCard",
  "params": {
    "shopUid": "shop-001",
    "characterUid": "char-42",
    "type": "Inventory",
    "deliveryMethod": "Inventory"
  },
  "id": 1
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "card": {
      "uid": "buy-card-9f3a",
      "shopUid": "shop-001",
      "characterUid": "char-42",
      "positions": [],
      "deliveryMethod": "Inventory",
      "type": "Inventory",
      "buyFailReasons": ["EmptyBuyCard"],
      "moneyShortfall": 0,
      "totalSum": 0
    }
  },
  "id": 1
}
```

**Пример ответа — отказ (корзина уже существует)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CannotCreateBuyCardAlreadyExists",
    "card": null
  },
  "id": 1
}
```

**Пример ответа — отказ (несовместимая доставка)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CannotUseDeliveryMethodForBuyCardType",
    "card": null
  },
  "id": 1
}
```

### recreateBuyCard

Пересоздаёт корзину покупки: при наличии активной корзины у персонажа она удаляется вместе с позициями, затем создаётся новая. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#recreatebuycard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"recreateBuyCard"`

**`params`**

| Поле             | Тип    | Описание                                                       |
| ---------------- | ------ | -------------------------------------------------------------- |
| `shopUid`        | string | Идентификатор магазина                                         |
| `characterUid`   | string | Идентификатор персонажа                                        |
| `type`           | string | `BuyCardTypeEnum`: `"Vehicle"` или `"Inventory"`               |
| `deliveryMethod` | string | `DeliveryMethod`: `"Inventory"` или `"NearVehicleSpawnPosition"` |

Совместимость: для `type: "Inventory"` допустим только `deliveryMethod: "Inventory"`; для `type: "Vehicle"` — только `"NearVehicleSpawnPosition"`.

Проверки: тип корзины и способ доставки должны быть совместимы. Если у персонажа уже была корзина, она снимается до создания новой; в ответе — новый `card.uid` и пустой `positions`.

**`result` (`RecreateBuyCardResult`)**

| Поле         | Тип            | Описание                                                                                          |
| ------------ | -------------- | ------------------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                     |
| `failReason` | string \| null | При `Fail`: `"CannotUseDeliveryMethodForBuyCardType"`; при `Ok` — `null`                          |
| `card`       | object \| null | При `Ok`: объект `BuyCard`; при `Fail` — `null`                                                   |

Структура `BuyCard` в `result.card` — как у [createBuyCard](#createbuycard).

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "recreateBuyCard",
  "params": {
    "shopUid": "shop-002",
    "characterUid": "char-42",
    "type": "Vehicle",
    "deliveryMethod": "NearVehicleSpawnPosition"
  },
  "id": 2
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "card": {
      "uid": "buy-card-b1c4",
      "shopUid": "shop-002",
      "characterUid": "char-42",
      "positions": [],
      "deliveryMethod": "NearVehicleSpawnPosition",
      "type": "Vehicle",
      "buyFailReasons": ["EmptyBuyCard"],
      "moneyShortfall": 0,
      "totalSum": 0
    }
  },
  "id": 2
}
```

**Пример ответа — отказ (несовместимая доставка)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CannotUseDeliveryMethodForBuyCardType",
    "card": null
  },
  "id": 2
}
```

### getActiveBuyCard

Возвращает активную корзину покупки персонажа с пересчитанными `buyFailReasons`, `moneyShortfall` и `totalSum`. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#getactivebuycard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"getActiveBuyCard"`

**`params`**

| Поле           | Тип    | Описание                |
| -------------- | ------ | ----------------------- |
| `characterUid` | string | Идентификатор персонажа |

**`result` (`GetActiveBuyCardResult`)**

| Поле         | Тип            | Описание                                                              |
| ------------ | -------------- | --------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                           |
| `failReason` | string \| null | При `Fail`: `"NoActiveBuyCard"`; при `Ok` — `null`                    |
| `card`       | object \| null | При `Ok`: объект `BuyCard`; при `Fail` — `null`                       |

Структура `BuyCard` в `result.card` — как у [createBuyCard](#createbuycard).

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "getActiveBuyCard",
  "params": {
    "characterUid": "char-42"
  },
  "id": 3
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "card": {
      "uid": "buy-card-9f3a",
      "shopUid": "shop-001",
      "characterUid": "char-42",
      "positions": [
        {
          "uid": "buy-pos-7c2a",
          "shopProductUid": "prod-ak74",
          "name": "AK-74",
          "prefabName": "AK74",
          "quantity": 3,
          "lineSum": 15000,
          "buyFailReasons": []
        }
      ],
      "deliveryMethod": "Inventory",
      "type": "Inventory",
      "buyFailReasons": [],
      "moneyShortfall": 0,
      "totalSum": 15000
    }
  },
  "id": 3
}
```

### addToBuyCard

Добавляет товар в корзину покупки. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#addtobuycard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"addToBuyCard"`

**`params`**

| Поле             | Тип    | Описание                                      |
| ---------------- | ------ | --------------------------------------------- |
| `buyCardUid`     | string | Идентификатор корзины покупки                 |
| `shopProductUid` | string | `uid` предложения из каталога магазина        |
| `quantity`       | number | Добавляемое количество (целое, > 0)           |

После успеха клиент перезапрашивает корзину через `getActiveBuyCard`. При нехватке на складе — отказ целиком, без частичного добавления.

**`result` (`AddToBuyCardResult`)**

| Поле          | Тип            | Описание                                                                                   |
| ------------- | -------------- | ------------------------------------------------------------------------------------------ |
| `status`      | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                               |
| `failReason`  | string \| null | При `Fail`: см. `AddToBuyCardFailReasonEnum`; при `Ok` — `null`                            |
| `positionUid` | string \| null | При `Ok`: `uid` созданной позиции; при `Fail` — `null`                   |

Причины отказа (`failReason`): `CannotAddShopProductNotFound`, `CannotAddShopProductNotInShop`, `CannotAddShopProductNotAvailableForBuy`, `CannotAddShopProductAlreadyInBuyCard`, `CannotAddIncompatibleBuyCardType`, `CannotUseNonPositiveQuantity`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "addToBuyCard",
  "params": {
    "buyCardUid": "buy-card-9f3a",
    "shopProductUid": "prod-ak74",
    "quantity": 2
  },
  "id": 4
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "positionUid": "buy-pos-7c2a"
  },
  "id": 4
}
```

### changeBuyCardPositionQuantity

Изменяет количество в позиции корзины покупки. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#changebuycardpositionquantity), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"changeBuyCardPositionQuantity"`

**`params`**

| Поле                 | Тип    | Описание                                |
| -------------------- | ------ | --------------------------------------- |
| `buyCardPositionUid` | string | Идентификатор позиции в корзине покупки |
| `newQuantity`        | number | Новое количество (целое, > 0)           |

При превышении остатка на складе `quantity` откатывается к допустимому значению, возвращается `Fail`. Актуальное состояние — через `getActiveBuyCard` после ответа.

**`result` (`ChangeBuyCardPositionQuantityResult`)**

| Поле         | Тип            | Описание                                                                                                                                 |
| ------------ | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                                                              |
| `failReason` | string \| null | При `Fail`: см. `ChangeBuyCardPositionQuantityFailReasonEnum`; при `Ok` — `null`                                                         |

Причины отказа (`failReason`): `CannotUseNonPositiveQuantity`, `InsufficientStock`, `ShopProductNotAvailableForBuy`, `ShopProductNotFound`, `ShopProductNotInShop`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "changeBuyCardPositionQuantity",
  "params": {
    "buyCardPositionUid": "buy-pos-7c2a",
    "newQuantity": 10
  },
  "id": 5
}
```

**Пример ответа — отказ (недостаточно на складе)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "InsufficientStock"
  },
  "id": 5
}
```

### removeBuyCardPosition

Удаляет позицию из корзины покупки. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#removebuycardposition).

**`method`:** `"removeBuyCardPosition"`

**`params`**

| Поле                 | Тип    | Описание                                |
| -------------------- | ------ | --------------------------------------- |
| `buyCardPositionUid` | string | Идентификатор позиции в корзине покупки |

**`result`:** `null`. После вызова клиент перезапрашивает корзину через `getActiveBuyCard`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "removeBuyCardPosition",
  "params": {
    "buyCardPositionUid": "buy-pos-7c2a"
  },
  "id": 6
}
```

### removeBuyCard

Удаляет корзину покупки. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#removebuycard).

**`method`:** `"removeBuyCard"`

**`params`**

| Поле         | Тип    | Описание                      |
| ------------ | ------ | ----------------------------- |
| `buyCardUid` | string | Идентификатор корзины покупки |

**`result`:** `null`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "removeBuyCard",
  "params": {
    "buyCardUid": "buy-card-9f3a"
  },
  "id": 7
}
```

### buy

Выполняет покупку. Подробнее: [Architecture/TradeManager.BuyMethods.md](../Architecture/TradeManager.BuyMethods.md#buy), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"buy"`

**`params`**

| Поле         | Тип    | Описание                      |
| ------------ | ------ | ----------------------------- |
| `buyCardUid` | string | Идентификатор корзины покупки |

Возвращает только статус. При `Fail` клиент перезапрашивает корзину через `getActiveBuyCard` для получения причин.

**`result` (`BuyStatus`)**

| Поле     | Тип    | Описание                                    |
| -------- | ------ | ------------------------------------------- |
| `status` | string | `"Ok"` или `"Fail"` (`OperationStatusEnum`) |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "buy",
  "params": {
    "buyCardUid": "buy-card-9f3a"
  },
  "id": 8
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok"
  },
  "id": 8
}
```

**Пример ответа — отказ**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail"
  },
  "id": 8
}
```

### createSellCard

Создаёт корзину продажи для персонажа в указанном магазине. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#createsellcard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"createSellCard"`

**`params`**

| Поле           | Тип    | Описание                |
| -------------- | ------ | ----------------------- |
| `shopUid`      | string | Идентификатор магазина  |
| `characterUid` | string | Идентификатор персонажа |

Проверки: у персонажа не должно быть другой активной корзины продажи.

**`result` (`CreateSellCardResult`)**

| Поле         | Тип            | Описание                                                                    |
| ------------ | -------------- | --------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                 |
| `failReason` | string \| null | При `Fail`: `"CannotCreateSellCardAlreadyExists"`; при `Ok` — `null`        |
| `card`       | object \| null | При `Ok`: объект `SellCard`; при `Fail` — `null`                            |

**`SellCard` (в `result.card`)**

| Поле           | Тип    | Описание                                         |
| -------------- | ------ | ------------------------------------------------ |
| `uid`          | string | Идентификатор корзины                            |
| `shopUid`      | string | Идентификатор магазина                           |
| `characterUid` | string | Идентификатор персонажа                          |
| `positions`       | array  | Позиции (`SellCardPosition`); при создании — `[]` |
| `sellFailReasons` | array  | Причины отказа на уровне корзины (`SellCardFailReasonEnum`); при пустой корзине — `["EmptySellCard"]` |
| `totalSum`        | number | Сумма `lineSum` по всем позициям; при пустой корзине — `0` |

Элемент `positions`: объекты с полями `uid`, `name`, `prefabName`, `entityUids`, `quantity`, `unitPrice`, `lineSum`, `sellFailReasons`, `allowedShopUids` (числа: `quantity`, `unitPrice`, `lineSum`; `entityUids`, `sellFailReasons` и `allowedShopUids` — массивы строк). Клиент получает `entityUids` в ответе, но не передаёт их в методы добавления или изменения количества.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "createSellCard",
  "params": {
    "shopUid": "shop-001",
    "characterUid": "char-42"
  },
  "id": 3
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "card": {
      "uid": "sell-card-3a9f",
      "shopUid": "shop-001",
      "characterUid": "char-42",
      "positions": [],
      "sellFailReasons": ["EmptySellCard"],
      "totalSum": 0
    }
  },
  "id": 3
}
```

**Пример ответа — отказ (корзина уже существует)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CannotCreateSellCardAlreadyExists",
    "card": null
  },
  "id": 3
}
```

### recreateSellCard

Пересоздаёт корзину продажи: при наличии активной корзины у персонажа она удаляется вместе с позициями, затем создаётся новая. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#recreatesellcard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"recreateSellCard"`

**`params`**

| Поле           | Тип    | Описание                |
| -------------- | ------ | ----------------------- |
| `shopUid`      | string | Идентификатор магазина  |
| `characterUid` | string | Идентификатор персонажа |

Проверки: если у персонажа уже была корзина, она снимается до создания новой; в ответе — новый `card.uid` и пустой `positions`.

**`result` (`RecreateSellCardResult`)**

| Поле     | Тип            | Описание                                      |
| -------- | -------------- | --------------------------------------------- |
| `status` | string         | `"Ok"` (`OperationStatusEnum`)                |
| `card`   | object \| null | При `Ok`: объект `SellCard`; при `Fail` — `null` |

Структура `SellCard` в `result.card` — как у [createSellCard](#createsellcard).

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "recreateSellCard",
  "params": {
    "shopUid": "shop-002",
    "characterUid": "char-42"
  },
  "id": 4
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "card": {
      "uid": "sell-card-c7d1",
      "shopUid": "shop-002",
      "characterUid": "char-42",
      "positions": [],
      "sellFailReasons": ["EmptySellCard"],
      "totalSum": 0
    }
  },
  "id": 4
}
```

### getActiveSellCard

Возвращает текущую активную корзину продажи персонажа. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#getactivesellcard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"getActiveSellCard"`

**`params`**

| Поле           | Тип    | Описание                |
| -------------- | ------ | ----------------------- |
| `characterUid` | string | Идентификатор персонажа |

**`result` (`GetActiveSellCardResult`)**

| Поле         | Тип            | Описание                                                             |
| ------------ | -------------- | -------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                          |
| `failReason` | string \| null | При `Fail`: `"NoActiveSellCard"`; при `Ok` — `null`                  |
| `card`       | object \| null | При `Ok`: объект `SellCard`; при `Fail` — `null`                     |

Структура `SellCard` в `result.card` — как у [createSellCard](#createsellcard).

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "getActiveSellCard",
  "params": {
    "characterUid": "char-42"
  },
  "id": 5
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "card": {
      "uid": "sell-card-3a9f",
      "shopUid": "shop-001",
      "characterUid": "char-42",
      "positions": [
        {
          "uid": "sell-pos-1b2c",
          "name": "AK-74",
          "prefabName": "AK74",
          "entityUids": ["entity-ak74-1", "entity-ak74-2"],
          "quantity": 2,
          "unitPrice": 4000,
          "lineSum": 8000,
          "sellFailReasons": [],
          "allowedShopUids": []
        }
      ],
      "sellFailReasons": [],
      "totalSum": 8000
    }
  },
  "id": 5
}
```

**Пример ответа — отказ (нет активной корзины)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "NoActiveSellCard",
    "card": null
  },
  "id": 5
}
```

### getInventoryForSell

Возвращает агрегированный снимок инвентаря персонажа для UI продажи. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#getinventoryforsell), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"getInventoryForSell"`

**`params`**

| Поле                           | Тип    | Описание                                                                                      |
| ------------------------------ | ------ | --------------------------------------------------------------------------------------------- |
| `characterUid`                 | string | Идентификатор персонажа                                                                       |
| `shopUid`                      | string | Идентификатор магазина (контекст цен и правил продажи)                                        |
| `isExcludeNotAvailableForSell` | bool   | Если `true` — только строки, доступные к продаже в этом магазине; если `false` — все строки   |

**`result` (`GetInventoryForSellResult`)**

| Поле               | Тип            | Описание                                                                                  |
| ------------------ | -------------- | ----------------------------------------------------------------------------------------- |
| `status`           | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                               |
| `failReason`       | string \| null | При `Fail`: `"ShopNotFound"` или `"CharacterNotFound"`; при `Ok` — `null`                 |
| `inventoryForSell` | object \| null | При `Ok`: объект `InventoryForSell`; при `Fail` — `null`                                |

**`InventoryForSell` (в `result.inventoryForSell`)**

| Поле         | Тип    | Описание                                      |
| ------------ | ------ | --------------------------------------------- |
| `positions`  | array  | Агрегированные строки (`InventoryProductPosition`) |
| `totalAmount`| number | Сумма `lineSum` по всем строкам в `positions` |

Элемент `positions`: объекты с полями `uid`, `name`, `prefabName`, `quantity`, `unitPrice`, `lineSum`, `sellFailReasons`, `allowedShopUids`. Числа: `quantity`, `unitPrice`, `lineSum`; `sellFailReasons` и `allowedShopUids` — массивы строк. Пустой `sellFailReasons` означает, что строка доступна для продажи в текущем контексте.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "getInventoryForSell",
  "params": {
    "characterUid": "char-42",
    "shopUid": "shop-001",
    "isExcludeNotAvailableForSell": true
  },
  "id": 6
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "inventoryForSell": {
      "positions": [
        {
          "uid": "inv-pos-ak74",
          "name": "AK-74",
          "prefabName": "AK74",
          "quantity": 3,
          "unitPrice": 4000,
          "lineSum": 12000,
          "sellFailReasons": [],
          "allowedShopUids": []
        }
      ],
      "totalAmount": 12000
    }
  },
  "id": 6
}
```

### addToSellCard

Добавляет в корзину продажи указанное количество из агрегированной строки инвентаря. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#addtosellcard), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"addToSellCard"`

**`params`**

| Поле                   | Тип    | Описание                                                              |
| ---------------------- | ------ | --------------------------------------------------------------------- |
| `sellCardUid`          | string | Идентификатор корзины продажи                                         |
| `inventoryPositionUid` | string | `uid` строки из `getInventoryForSell` → `positions`                 |
| `quantity`             | number | Добавляемое количество (целое, > 0)                                   |

После успеха клиент перезапрашивает корзину через `getActiveSellCard`. При превышении инвентаря — отказ целиком, без частичного добавления.

**`result` (`AddToSellCardResult`)**

| Поле              | Тип            | Описание                                                                                                                                 |
| ----------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `status`          | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                                                              |
| `failReason`      | string \| null | При `Fail`: см. `AddToSellCardFailReasonEnum`; при `Ok` — `null`                                                                         |
| `positionUid`     | string \| null | При `Ok`: `uid` созданной или обновлённой строки корзины; при `Fail` — `null`                                                            |
| `allowedShopUids` | array \| null  | При `failReason = "CannotSellEntityInShop"`: список магазинов, где продажа разрешена; иначе — `null` или не используется                 |

Причины отказа (`failReason`): `CannotAddInventoryPositionNotFound`, `CannotAddQuantityExceedsInventory`, `CannotUseNonPositiveQuantity`, `CannotSellEntity`, `CannotSellEntityInShop`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "addToSellCard",
  "params": {
    "sellCardUid": "sell-card-3a9f",
    "inventoryPositionUid": "inv-pos-ak74",
    "quantity": 2
  },
  "id": 7
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "positionUid": "sell-pos-1b2c",
    "allowedShopUids": null
  },
  "id": 7
}
```

### changeSellCardPositionQuantity

Изменяет количество в строке корзины продажи. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#changesellcardpositionquantity), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"changeSellCardPositionQuantity"`

**`params`**

| Поле                  | Тип    | Описание                                      |
| --------------------- | ------ | --------------------------------------------- |
| `sellCardPositionUid` | string | Идентификатор позиции в корзине продажи       |
| `newQuantity`         | number | Новое количество (целое, > 0)                 |

При превышении инвентаря `quantity` откатывается к допустимому значению, возвращается `Fail`. Актуальное состояние — через `getActiveSellCard` после ответа.

**`result` (`ChangeSellCardPositionQuantityResult`)**

| Поле         | Тип            | Описание                                                                                          |
| ------------ | -------------- | ------------------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                       |
| `failReason` | string \| null | При `Fail`: см. `ChangeSellCardPositionQuantityFailReasonEnum`; при `Ok` — `null` |

Причины отказа (`failReason`): `CannotUseNonPositiveQuantity`, `QuantityExceedsInventory`, `CannotSellEntity`, `CannotSellEntityInShop`, `CannotSellItemNotOwnedByCharacter`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "changeSellCardPositionQuantity",
  "params": {
    "sellCardPositionUid": "sell-pos-1b2c",
    "newQuantity": 1
  },
  "id": 8
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null
  },
  "id": 8
}
```

**Пример ответа — отказ (превышение инвентаря)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "QuantityExceedsInventory"
  },
  "id": 8
}
```

### removeSellCardPosition

Удаляет позицию из корзины продажи. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#removesellcardposition).

**`method`:** `"removeSellCardPosition"`

**`params`**

| Поле                  | Тип    | Описание                                |
| --------------------- | ------ | --------------------------------------- |
| `sellCardPositionUid` | string | Идентификатор позиции в корзине продажи |

**`result`:** `null`. После вызова клиент перезапрашивает корзину через `getActiveSellCard`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "removeSellCardPosition",
  "params": {
    "sellCardPositionUid": "sell-pos-1b2c"
  },
  "id": 9
}
```

### removeSellCard

Удаляет корзину продажи. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#removesellcard).

**`method`:** `"removeSellCard"`

**`params`**

| Поле          | Тип    | Описание                    |
| ------------- | ------ | --------------------------- |
| `sellCardUid` | string | Идентификатор корзины продажи |

**`result`:** `null`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "removeSellCard",
  "params": {
    "sellCardUid": "sell-card-3a9f"
  },
  "id": 10
}
```

### sell

Выполняет продажу всех позиций корзины. Подробнее: [Architecture/TradeManager.SellMethods.md](../Architecture/TradeManager.SellMethods.md#sell), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"sell"`

**`params`**

| Поле          | Тип    | Описание                    |
| ------------- | ------ | --------------------------- |
| `sellCardUid` | string | Идентификатор корзины продажи |

Возвращает только статус. При `Fail` клиент перезапрашивает корзину через `getActiveSellCard` для получения причин.

**`result` (`SellResult`)**

| Поле     | Тип    | Описание                                    |
| -------- | ------ | ------------------------------------------- |
| `status` | string | `"Ok"` или `"Fail"` (`OperationStatusEnum`) |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "sell",
  "params": {
    "sellCardUid": "sell-card-3a9f"
  },
  "id": 11
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok"
  },
  "id": 11
}
```

**Пример ответа — отказ**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail"
  },
  "id": 11
}
```

### getCash

Возвращает сумму наличных у персонажа. Подробнее: [Architecture/BankingManager.md](../Architecture/BankingManager.md#getcash), сущности: [Architecture/BankingManager.Entities.md](../Architecture/BankingManager.Entities.md).

**`method`:** `"banking@getCash"`

**`params`**

| Поле           | Тип    | Описание              |
| -------------- | ------ | --------------------- |
| `characterUid` | string | Идентификатор персонажа |

**`result` (`GetCashResult`)**

| Поле         | Тип            | Описание                                                          |
| ------------ | -------------- | ----------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                       |
| `failReason` | string \| null | При `Fail`: `"CharacterNotFound"`; при `Ok` — `null`              |
| `amount`     | number \| null | При `Ok`: сумма наличных в минимальных денежных единицах; при `Fail` — `null` |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "banking@getCash",
  "params": {
    "characterUid": "char-42"
  },
  "id": 12
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "amount": 15000
  },
  "id": 12
}
```

**Пример ответа — отказ (персонаж не найден)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "CharacterNotFound",
    "amount": null
  },
  "id": 12
}
```

### getDescriptionByPrefab

Возвращает текстовое описание товара по имени префаба. Подробнее: [Architecture/TradeManager.ShopMethods.md](../Architecture/TradeManager.ShopMethods.md#getdescriptionbyprefab), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"catalog@getDescriptionByPrefab"`

**`params`**

| Поле         | Тип    | Описание              |
| ------------ | ------ | --------------------- |
| `prefabName` | string | Имя префаба товара    |

**`result` (`GetDescriptionByPrefabResult`)**

| Поле          | Тип            | Описание                                                          |
| ------------- | -------------- | ----------------------------------------------------------------- |
| `status`      | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                       |
| `failReason`  | string \| null | При `Fail`: `"PrefabNotFound"`; при `Ok` — `null`                 |
| `description` | string \| null | При `Ok`: текст описания; при `Fail` — `null`                     |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "catalog@getDescriptionByPrefab",
  "params": {
    "prefabName": "AK74"
  },
  "id": 9
}
```

**Пример ответа — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "description": "Автомат Калашникова образца 1974 года. Стандартное стрелковое оружие."
  },
  "id": 9
}
```

**Пример ответа — отказ (префаб не найден)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "PrefabNotFound",
    "description": null
  },
  "id": 9
}
```
