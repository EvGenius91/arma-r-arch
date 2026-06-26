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

Элемент `positions`: объекты с полями `uid`, `name`, `prefabName`, `entityUids` (массив строк), `quantity`, `lineSum` (числа: `quantity`, `lineSum`).

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
          "entityUids": ["ent-101", "ent-102"],
          "quantity": 2,
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
