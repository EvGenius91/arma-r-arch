# Правило разделения Service и API

Назад: [[ArchBackend.md]]

## Назначение

Единое правило для всех сервисов бекенда (TradeService, BankingService, EntityService и др.): **сервис работает с доменом, API — с транспортным контрактом RPC**.

---

## Слои

### Service (домен)

- Реализует `*ServiceInterface` из `Contracts/{ServiceName}/`.
- Возвращает **доменные DTO** (`BuyCardDto`, `ShopProductDto`, `string` для `positionUid`, `void` где нет полезной нагрузки).
- При бизнес-отказе бросает **типизированное доменное исключение** из `Contracts/{ServiceName}/Exceptions/`.
- **Не** возвращает `*ResultDto`, **не** использует `OperationStatus`, **не** знает про JSON-RPC/HTTP.

### API (транспорт)

- Принимает JSON-RPC `params`, валидирует вход.
- Вызывает метод сервиса.
- Формирует **`*ResultDto`** по контракту из документации/API.
- При успехе: `status = Ok`, `failReason = null`, доменные данные в полях результата.
- При `*ServiceException`: `status = Fail`, `failReason` из `$e->getFailReason()`, данные = `null` / пустые значения.
- Сериализует `*ResultDto` в JSON-RPC `result`.

---

## Универсальная формула

```
API Handler:
  try:
    data = Service.method(params...)
    return ResultDto(Ok, failReason: null, ...data)
  catch ServiceException e:
    return ResultDto(Fail, failReason: e.getFailReason(), ...nulls)
```

**Сервис:** `return Dto` или `throw Exception`  
**API:** `ResultDto(Ok | Fail)`

---

## Исключения

| Тип | Где возникает | Как обрабатывает API |
|-----|---------------|----------------------|
| Validation (нет поля, неверный тип) | API, до вызова сервиса | JSON-RPC transport error / HTTP 4xx, **не** `failReason` |
| Domain (`*ServiceException`) | Service | `status = Fail`, `failReason` из исключения |
| Infrastructure (БД, сеть) | Service / инфра | проброс выше, **не** маппить в `failReason` |

Базовый класс:

```php
abstract class TradeServiceException extends \RuntimeException
{
    abstract public function getFailReason(): \BackedEnum;
}
```

Одно исключение = одна причина отказа из документации (`NoActiveBuyCard`, `ShopNotFound`, …).

---

## Что где лежит

| Артефакт | Директория | Кто создаёт |
|----------|------------|-------------|
| `*ServiceInterface` | `Contracts/{ServiceName}/` | — |
| Доменные DTO (`BuyCardDto`, …) | `Contracts/{ServiceName}/Dto/` | Service (return) |
| Enum причин отказа | `Contracts/{ServiceName}/Enums/` | Exception + Result |
| Доменные исключения | `Contracts/{ServiceName}/Exceptions/` | Service (throw) |
| `*ResultDto` | `Contracts/{ServiceName}/Dto/` или `Http/JsonRpc/Dto/` | **только API** |
| `OperationStatus` | `Contracts/Common/Enums/` | **только API** |
| Eloquent, репозитории, билдеры | `Models/`, `Services/` | Service (внутренне) |

---

## Правила по типам методов

### 1. Метод возвращает сущность

**Пример:** `getActiveBuyCard`, `createBuyCard`, `getShopCategories`

| | Service | API |
|---|---------|-----|
| Успех | `return BuyCardDto` / `ShopCategoryDto[]` | `ResultDto(Ok, null, card/categories)` |
| Отказ | `throw NoActiveBuyCardException` | `ResultDto(Fail, NoActiveBuyCard, null)` |

### 2. Метод возвращает примитив / id

**Пример:** `addToBuyCard` → `positionUid`

| | Service | API |
|---|---------|-----|
| Успех | `return string` (positionUid) | `ResultDto(Ok, null, positionUid: ...)` |
| Отказ | `throw CannotAddShopProductNotFoundException` | `ResultDto(Fail, failReason, positionUid: null)` |

### 3. Метод без полезной нагрузки при успехе

**Пример:** `removeBuyCard`, `removeBuyCardPosition`, `changeBuyCardPositionQuantity` (только status)

| | Service | API |
|---|---------|-----|
| Успех | `return void` | `ResultDto(Ok, null)` или `null` (как в API-доке) |
| Отказ | `throw ...Exception` | `ResultDto(Fail, failReason)` |

### 4. Метод возвращает только статус без failReason

**Пример:** `buy`, `sell` — в доке при Fail нет `failReason`, клиент перезапрашивает корзину

| | Service | API |
|---|---------|-----|
| Успех | `return void` | `{ "status": "Ok" }` |
| Отказ | `throw BuyFailedException` (без публичного failReason в RPC) | `{ "status": "Fail" }` |

Сервис может бросать исключение для отката транзакции; API **не** добавляет `failReason`, если его нет в контракте метода.

### 5. Метод с частичным failReason

**Пример:** `addToSellCard` — при `CannotSellEntityInShop` ещё `allowedShopUids`

| | Service | API |
|---|---------|-----|
| Успех | `return AddToSellCardSuccessDto(positionUid)` | `ResultDto(Ok, null, positionUid, allowedShopUids: null)` |
| Отказ | `throw CannotSellEntityInShopException(allowedShopUids: [...])` | `ResultDto(Fail, failReason, positionUid: null, allowedShopUids: e.getAllowedShopUids())` |

Дополнительные поля при Fail — в исключении, API копирует их в Result.

---

## Запреты

- Service **не** возвращает `*ResultDto`.
- Service **не** возвращает `OperationStatus`.
- API **не** содержит бизнес-логики (проверки склада, денег, владения).
- API **не** ловит domain exception и не «чинит» — только маппит в Result.
- Доменные DTO **не** дублируют поля `status` / `failReason`.

---

## Чеклист для нового метода

1. Описать контракт RPC: `*ResultDto`, поля, enum `*FailReason`.
2. В интерфейсе сервиса: return = доменный тип(ы), `@throws` = список `*Exception`.
3. Для каждого `failReason` из документации — свой класс исключения с `getFailReason()`.
4. Handler: validate → try service → Result(Ok) / catch → Result(Fail).
5. Mapper: ResultDto → массив JSON-RPC.

---

## Пример (эталон)

**Service:**

```php
public function getActiveBuyCard(string $characterUid): BuyCardDto;
// throw NoActiveBuyCardException
```

**API:**

```php
try {
    return new GetActiveBuyCardResultDto(
        OperationStatus::Ok,
        failReason: null,
        card: $this->tradeService->getActiveBuyCard($characterUid),
    );
} catch (TradeServiceException $e) {
    return new GetActiveBuyCardResultDto(
        OperationStatus::Fail,
        failReason: $e->getFailReason(),
        card: null,
    );
}
```

---

**Кратко:** сервис говорит «вот данные» или «ошибка домена»; API переводит это в «Ok/Fail + failReason» для JSON-RPC.
