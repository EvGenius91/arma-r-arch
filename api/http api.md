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

### getInventoryByCharacterUid

Возвращает плоский список сущностей инвентаря / носимого набора персонажа по `characterUid`. Не агрегат Trade (`getInventoryForSell`): вложенность — через `parentContainerUid` / `inventorySlotUid`. Пустой список — успех. Подробнее: [Architecture/EntityManager.HttpMethods.md](../Architecture/EntityManager.HttpMethods.md#getinventorybycharacteruid), сущности: [Architecture/EntityManager.Entities.md](../Architecture/EntityManager.Entities.md).

**`method`:** `"entity@getInventoryByCharacterUid"`

**`params`**

| Поле           | Тип    | Описание                |
| -------------- | ------ | ----------------------- |
| `characterUid` | string | Идентификатор персонажа |

**`result` (`GetInventoryByCharacterUidResult`)**

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`); для этого метода — `"Ok"`   |
| `failReason` | string \| null | При `Ok` — `null`; доменных Fail нет                                     |
| `entities`   | array \| null  | При `Ok`: массив `EntityItem` (может быть пустым); при `Fail` — `null`   |

Элемент `entities` (`EntityItem`):

| Поле                 | Тип            | Описание                                      |
| -------------------- | -------------- | --------------------------------------------- |
| `uid`                | string         | Идентификатор сущности                        |
| `prefabName`         | string         | Имя префаба                                   |
| `isContainer`        | boolean        | Признак контейнера                            |
| `position`           | object \| null | Координаты в мире `{ x, y, z }` или `null`    |
| `angle`              | object \| null | Угол в мире `{ x, y, z }` или `null`          |
| `parentContainerUid` | string \| null | Родительский контейнер                        |
| `inventorySlotUid`   | string \| null | Слот инвентаря / экипировки                   |
| `ownerCharacterUid`  | string \| null | Персонаж-владелец                             |
| `storageType`        | string         | Тип хранилища (`StorageTypeEnum`, не null)    |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "entity@getInventoryByCharacterUid",
  "params": {
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
    "entities": [
      {
        "uid": "ent-backpack-1",
        "prefabName": "Backpack_01",
        "isContainer": true,
        "position": null,
        "angle": null,
        "parentContainerUid": null,
        "inventorySlotUid": "slot-backpack",
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      },
      {
        "uid": "ent-can-001",
        "prefabName": "Food_Can",
        "isContainer": false,
        "position": null,
        "angle": null,
        "parentContainerUid": "ent-backpack-1",
        "inventorySlotUid": null,
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      }
    ]
  },
  "id": 1
}
```

**Пример ответа — успех (пустой инвентарь)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "entities": []
  },
  "id": 1
}
```

### findEntitiesByUidList

Возвращает сущности реестра по списку uid вместе с вложенными потомками (плоский список по `parentContainerUid`). Отсутствующие uid пропускаются. Сначала найденные из `uidList` в порядке запроса, затем потомки. Пустой `uidList` — успех с пустым `entities`. Подробнее: [Architecture/EntityManager.HttpMethods.md](../Architecture/EntityManager.HttpMethods.md#findentitiesbyuidlist), сущности: [Architecture/EntityManager.Entities.md](../Architecture/EntityManager.Entities.md).

**`method`:** `"entity@findEntitiesByUidList"`

**`params`**

| Поле      | Тип      | Описание                         |
| --------- | -------- | -------------------------------- |
| `uidList` | string[] | Список идентификаторов сущностей |

**`result` (`FindEntitiesByUidListResult`)**

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`); при успехе сервиса — `"Ok"` |
| `failReason` | string \| null | При `Ok` — `null`; при Fail — `"InvalidParams"`                          |
| `entities`   | array \| null  | При `Ok`: плоский массив `EntityItem` — запрошенные uid и потомки (может быть пустым); при `Fail` — `null` |

Элемент `entities` (`EntityItem`) — те же поля, что у `getInventoryByCharacterUid`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "entity@findEntitiesByUidList",
  "params": {
    "uidList": ["ent-backpack-1", "ent-missing"]
  },
  "id": 2
}
```

**Пример ответа — успех (рюкзак и содержимое; отсутствующий uid пропущен)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "entities": [
      {
        "uid": "ent-backpack-1",
        "prefabName": "Backpack_01",
        "isContainer": true,
        "position": null,
        "angle": null,
        "parentContainerUid": null,
        "inventorySlotUid": "slot-backpack",
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      },
      {
        "uid": "ent-can-001",
        "prefabName": "Food_Can",
        "isContainer": false,
        "position": null,
        "angle": null,
        "parentContainerUid": "ent-backpack-1",
        "inventorySlotUid": null,
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      }
    ]
  },
  "id": 2
}
```

**Пример ответа — успех (пустой uidList)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "entities": []
  },
  "id": 2
}
```

### applyOperations (EntityManager)

Принимает упорядоченную пачку операций расположения сущностей от EntityManager при flush. Вызывается только с игрового сервера (не из UI / CSM). Пачка не атомарна по всем `entityUid`; успех/отказ — по каждой ops. Подробнее: [Architecture/EntityManager.HttpMethods.md](../Architecture/EntityManager.HttpMethods.md), очередь: [Architecture/EntityManager.Operations.md](../Architecture/EntityManager.Operations.md).

**`method`:** `"entity@applyOperations"`

**`params`**

| Поле         | Тип   | Описание                                                                 |
| ------------ | ----- | ------------------------------------------------------------------------ |
| `operations` | array | Непустой упорядоченный список `EntityOperation` (порядок = порядок apply) |

Элемент `operations`:

| Поле              | Тип    | Описание                                                                 |
| ----------------- | ------ | ------------------------------------------------------------------------ |
| `entityUid`       | string | Идентификатор сущности                                                   |
| `type`            | string | `PickUpEntity`, `DropEntity`, `MoveEntity`, `EntityTransformChanged`, `EquipItem`, `UnequipItem`, `SwapEquipment`, `TransferEntity`, `SplitStack`, `MergeStack`, `DestroyEntity` |
| `resetGeneration` | number | Поколение сущности на момент отправки                                    |
| `characterUid`    | string \| null | Персонаж-инициатор; `null` для `EntityTransformChanged`                  |
| `payload`         | object | Поля по `type` (см. ниже)                                                |

`payload` для основных типов (`storageType` обязателен для операций, изменяющих инвентарное расположение):

| `type` | Поля `payload` |
| ------ | -------------- |
| `PickUpEntity` / `MoveEntity` | `targetContainerUid` (string \| null; `null` для корневого слота экипировки), `slot` (string \| null; обязателен при `targetContainerUid` = null), `storageType` (string) |
| `DropEntity` | `position` `{ x, y, z }` (числа), `angle` `{ x, y, z }` (числа), `storageType` (string) |
| `EntityTransformChanged` | `position` `{ x, y, z }` (числа), `angle` `{ x, y, z }` (числа); без `storageType` |
| `TransferEntity` | `toCharacterUid` (string), `targetContainerUid` (string \| null), `slot` (string \| null), `storageType` (string) |

Остальные типы — см. [EntityManager.HttpMethods.md](../Architecture/EntityManager.HttpMethods.md).

**`result` (`ApplyEntityOperationsResult`)**

| Поле               | Тип            | Описание                                                                 |
| ------------------ | -------------- | ------------------------------------------------------------------------ |
| `status`           | string         | `"Ok"` — пачка обработана (в т.ч. с частичными Fail по ops); `"Fail"` — отказ на уровне запроса |
| `failReason`       | string \| null | При request Fail: `"EmptyOperations"`, `"InvalidParams"`; при `Ok` — `null` |
| `operationResults` | array \| null  | При `Ok`: массив той же длины и порядка, что `operations`; при Fail — `null` |

Элемент `operationResults`:

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"`                                                      |
| `failReason` | string \| null | При Fail: `"EntityNotFound"`, `"StaleAfterReset"`, `"ContainerNotFound"`, `"InvalidLocation"`, `"AlreadyTaken"`, `"InvalidOperation"`; при Ok — `null` |

Индексация 1:1 с входным `operations[]`.

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "entity@applyOperations",
  "params": {
    "operations": [
      {
        "entityUid": "ent-can-001",
        "type": "DropEntity",
        "resetGeneration": 0,
        "characterUid": "char-p1",
        "payload": {
          "position": { "x": 100.5, "y": 12.0, "z": -40.25 },
          "angle": { "x": 0.0, "y": 45.0, "z": 0.0 },
          "storageType": "SCR_CharacterInventoryStorageComponent"
        }
      },
      {
        "entityUid": "ent-can-001",
        "type": "PickUpEntity",
        "resetGeneration": 0,
        "characterUid": "char-p2",
        "payload": {
          "targetContainerUid": "cont-inv-p2",
          "slot": null,
          "storageType": "SCR_CharacterInventoryStorageComponent"
        }
      }
    ]
  },
  "id": 42
}
```

**Пример ответа — обе ops Ok**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "operationResults": [
      { "status": "Ok", "failReason": null },
      { "status": "Ok", "failReason": null }
    ]
  },
  "id": 42
}
```

**Пример ответа — частичный успех (AlreadyTaken)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "operationResults": [
      { "status": "Ok", "failReason": null },
      { "status": "Fail", "failReason": "AlreadyTaken" }
    ]
  },
  "id": 42
}
```

**Пример ответа — StaleAfterReset**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "operationResults": [
      { "status": "Fail", "failReason": "StaleAfterReset" }
    ]
  },
  "id": 42
}
```

**Пример ответа — отказ на уровне запроса (пустая пачка)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "EmptyOperations",
    "operationResults": null
  },
  "id": 42
}
```

### getPendingCommands

Возвращает пачку команд очереди бекенд → игровой сервер **текущей игровой сессии**. Команды других сессий не выдаются. При выдаче команды переходят в in-flight (повторный poll не отдаёт их до отчёта). Пустой `commands` — успех. Вызывается из `CommandBus::run` каждые 0.5 с. `serverSessionUid` в result не отдаётся. Подробнее: [CommandBus/CommandBus.HttpMethods.md](../CommandBus/CommandBus.HttpMethods.md#getpendingcommands), сущности: [CommandBus/CommandBus.Entities.md](../CommandBus/CommandBus.Entities.md).

**`method`:** `"command@getPendingCommands"`

**`params`**

Нет (пустой объект `{}`). Сервер игры — в контексте API-ключа.

**`result` (`GetPendingCommandsResult`)**

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`); при успехе сервиса — `"Ok"` |
| `failReason` | string \| null | При `Ok` — `null`                                                        |
| `commands`   | array \| null  | При `Ok`: массив `Command` (может быть пустым); при `Fail` — `null`      |

Элемент `commands` (`Command`):

| Поле      | Тип    | Описание                                      |
| --------- | ------ | --------------------------------------------- |
| `uid`     | string | Идентификатор команды (для отчёта)            |
| `type`    | string | `SpawnEntity` \| `DeleteEntity`               |
| `payload` | object | Параметры по `type`; для `SpawnEntity` и `DeleteEntity` — `{ "entityUid": string }` |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "command@getPendingCommands",
  "params": {},
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
    "commands": [
      {
        "uid": "cmd-001",
        "type": "SpawnEntity",
        "payload": {
          "entityUid": "ent-can-001"
        }
      },
      {
        "uid": "cmd-002",
        "type": "DeleteEntity",
        "payload": {
          "entityUid": "ent-backpack-1"
        }
      }
    ]
  },
  "id": 1
}
```

**Пример ответа — успех (пустая очередь)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "commands": []
  },
  "id": 1
}
```

### reportCommands

Принимает пачку отчётов о выполнении команд. Неизвестный / уже завершённый `commandUid` — идемпотентный no-op для записи. Пустой `reports` — успех. Вызывается из `CommandBus::reportCommands`. Подробнее: [CommandBus/CommandBus.HttpMethods.md](../CommandBus/CommandBus.HttpMethods.md#reportcommands), сущности: [CommandBus/CommandBus.Entities.md](../CommandBus/CommandBus.Entities.md).

**`method`:** `"command@reportCommands"`

**`params`**

| Поле      | Тип   | Описание                          |
| --------- | ----- | --------------------------------- |
| `reports` | array | Список `CommandReport` (может быть пустым) |

Элемент `reports` (`CommandReport`):

| Поле         | Тип            | Описание                                      |
| ------------ | -------------- | --------------------------------------------- |
| `commandUid` | string         | `Command.uid` из ранее полученной команды     |
| `status`     | string         | `"Completed"` или `"Fail"`                    |
| `failReason` | string \| null | При `Completed` — `null`; при `Fail` — обязателен |

**`result` (`ReportCommandsResult`)**

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                              |
| `failReason` | string \| null | При `Ok` — `null`; при request Fail — `"InvalidParams"`                  |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "command@reportCommands",
  "params": {
    "reports": [
      {
        "commandUid": "cmd-001",
        "status": "Completed",
        "failReason": null
      },
      {
        "commandUid": "cmd-002",
        "status": "Fail",
        "failReason": "CannotSpawn"
      }
    ]
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
    "failReason": null
  },
  "id": 2
}
```

**Пример ответа — отказ на уровне запроса (невалидные params)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "InvalidParams"
  },
  "id": 2
}
```

### gameStarted

Игровой сервер вызывает метод при старте. Бекенд закрывает предыдущие активные сессии (`stoppedAt = now()`), регистрирует новую `ServerSession` (`startedAt = now()`, `stoppedAt = null`) и ставит в очередь CommandBus команду `SpawnEntity` для каждой сущности в мире без владельца-персонажа и без родительского контейнера (вложенные не ставятся). Одновременно активна только одна сессия. Доменных отказов нет. Подробнее: [Architecture/GameManager.md](../Architecture/GameManager.md#gamestarted), сущности: [Architecture/GameManager.Entities.md](../Architecture/GameManager.Entities.md).

**`method`:** `"game@gameStarted"`

**`params`**

| Поле               | Тип    | Описание                                      |
| ------------------ | ------ | --------------------------------------------- |
| `serverSessionUid` | string | Идентификатор сессии игрового сервера |

Сервер игры — в контексте API-ключа.

**`result` (`GameStartedResult`)**

| Поле         | Тип            | Описание                                                                 |
| ------------ | -------------- | ------------------------------------------------------------------------ |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`); при успехе сервиса — `"Ok"` |
| `failReason` | string \| null | При `Ok` — `null`                                                        |

**Пример запроса**

```json
{
  "jsonrpc": "2.0",
  "method": "game@gameStarted",
  "params": {
    "serverSessionUid": "session-001"
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
    "failReason": null
  },
  "id": 1
}
```
