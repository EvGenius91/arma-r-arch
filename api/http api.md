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

Элемент `positions`: объекты с полями `uid`, `shopProductUid`, `name`, `prefabName`, `quantity`, `lineSum` (числа: `quantity`, `lineSum`).

Пример элемента `positions` (непустая корзина):

```json
{
  "uid": "buy-pos-7c2a",
  "shopProductUid": "prod-ak74",
  "name": "AK-74",
  "prefabName": "AK74",
  "quantity": 3,
  "lineSum": 15000
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
      "type": "Inventory"
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
      "type": "Vehicle"
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
| `positions`    | array  | Позиции (`SellCardPosition`); при создании — `[]` |

Элемент `positions`: объекты с полями `uid`, `name`, `prefabName`, `quantity`, `unitPrice`, `lineSum` (числа: `quantity`, `unitPrice`, `lineSum`). Поле `entityUids` может присутствовать во внутреннем составе строки, но клиент UI его не использует.

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
      "positions": []
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
      "positions": []
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
          "quantity": 2,
          "unitPrice": 4000,
          "lineSum": 8000
        }
      ]
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

Элемент `positions`: объекты с полями `uid`, `name`, `prefabName`, `quantity`, `unitPrice`, `lineSum`, `isAvailableForSell`, `sellFailReason` (при `isAvailableForSell = false`), `allowedShopUids` (при `sellFailReason = "CannotSellEntityInShop"`). Числа: `quantity`, `unitPrice`, `lineSum`.

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
          "isAvailableForSell": true
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

**`result` (`ChangeSellCardPositionQuantityResult`)**

| Поле         | Тип            | Описание                                                                                          |
| ------------ | -------------- | ------------------------------------------------------------------------------------------------- |
| `status`     | string         | `"Ok"` или `"Fail"` (`OperationStatusEnum`)                                                       |
| `failReason` | string \| null | При `Fail`: `"CannotUseNonPositiveQuantity"` или `"CannotAddQuantityExceedsInventory"`; при `Ok` — `null` |

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

### getDescriptionByPrefab

Возвращает текстовое описание товара по имени префаба. Подробнее: [Architecture/TradeManager.ShopMethods.md](../Architecture/TradeManager.ShopMethods.md#getdescriptionbyprefab), сущности: [Architecture/TradeManager.Entities.md](../Architecture/TradeManager.Entities.md).

**`method`:** `"getDescriptionByPrefab"`

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
  "method": "getDescriptionByPrefab",
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
