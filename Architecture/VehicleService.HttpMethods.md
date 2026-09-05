# HTTP-методы VehicleService (игра → бекенд)

Назад: [[VehicleService.md]] · [[VehicleService.Entities.md]] · [[Architecture.md]]

Документ описывает JSON-RPC контракт VehicleService на бекенде:

- отправка пачки операций техники — [[#applyOperations]].

Доменные методы без RPC: [[VehicleService.md#findVehicle]], [[VehicleService.md#registerVehicle]].  
Сущности операций: [[VehicleService.Entities.md]].  
Транспорт и примеры: [[../api/http api.md]].

```text
Game → vehicle@applyOperations(operations[]) → per-op Ok/Fail
```

---

## applyOperations

**JSON-RPC `method`:** `"vehicle@applyOperations"`

**Сигнатура**

```text
applyOperations(operations[]): ApplyVehicleOperationsResult
```

Доменный сервис (`VehicleServiceInterface`):

```text
applyOperations(list<VehicleOperationDto> vehicleOperations): list<VehicleOperationOutcomeDto>
```

**Аргументы**

- `operations: VehicleOperation[]` — непустой упорядоченный список операций. Порядок в массиве = порядок применения на бекенде.

### Элемент `VehicleOperation`

| Поле | Тип | Описание |
|------|-----|----------|
| `entityUid` | string | Идентификатор сущности техники (совпадает с `Vehicle.entityUid` / `EntityItem.uid`) |
| `type` | string | Тип операции (см. каталог ниже) |
| `driverCharacterUid` | string | Персонаж на месте водителя |

Поля без вложенного `payload`.

### Каталог `type`

| `type` | Смысл |
|--------|--------|
| `changeDriver` | персонаж занял место водителя; бекенд записывает `lastDriverCharacterUid` |

### Описание

1. Игровой сервер собирает операции техники в пачку с сохранением порядка и вызывает этот метод.
2. Бекенд применяет операции **по порядку** в `operations`.
3. Транзакция — на одну операцию, не на всю пачку.
4. Если для `entityUid` в этой пачке уже была отказавшая операция, следующие операции с тем же uid получают тот же `failReason` и не применяются.
5. Нет строки `vehicles` по `entityUid` — `VehicleNotFound`. Строка не создаётся.
6. Нет обработчика для `type` — `UnsupportedOperationType`.
7. `changeDriver` записывает `lastDriverCharacterUid` = `driverCharacterUid`.
8. Каждая операция логируется (успех и отказ).
9. Сервис не бросает доменных исключений по этому методу; API оборачивает ответ в `ApplyVehicleOperationsResult`. При невалидных params отказ — на уровне запроса, до вызова сервиса.
10. Неизвестный `type` (нет в enum) — `InvalidParams` на уровне запроса. Известный тип без обработчика — `UnsupportedOperationType` в `operationResults`.

### Результат (`ApplyVehicleOperationsResult`)

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | string | `Ok` — пачка обработана (в т.ч. с частичными Fail по ops); `Fail` — отказ на уровне запроса |
| `failReason` | string \| null | При request-level Fail — причина; иначе `null` |
| `operationResults` | VehicleOperationResult[] \| null | При `Ok`: массив той же длины и порядка, что `operations`; при request Fail — `null` |

Элемент `operationResults[]` (`VehicleOperationResult`):

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | string | `Ok` или `Fail` |
| `failReason` | string \| null | При Fail — причина; при Ok — `null` |

Индексация **1:1** с входным `operations[]` (без дублирования `entityUid` в результате — идентификация по индексу).

### Причины отказа на уровне запроса (`failReason`)

- `EmptyOperations` — массив `operations` пуст.
- `InvalidParams` — невалидная структура params / элемента ops до доменной обработки (в т.ч. неизвестный `type`).

### Причины отказа на уровне операции (`operationResults[].failReason`)

- `VehicleNotFound` — техники с этим `entityUid` нет; запись не создаётся.
- `UnsupportedOperationType` — тип операции не поддерживается (нет обработчика).

### Примеры

**Запрос**

```json
{
  "jsonrpc": "2.0",
  "method": "vehicle@applyOperations",
  "params": {
    "operations": [
      {
        "entityUid": "vehicle-001",
        "type": "changeDriver",
        "driverCharacterUid": "char-42"
      }
    ]
  },
  "id": 1
}
```

**Ответ — успех**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "operationResults": [
      { "status": "Ok", "failReason": null }
    ]
  },
  "id": 1
}
```

**Ответ — отказ операции (техника не найдена)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "operationResults": [
      { "status": "Fail", "failReason": "VehicleNotFound" }
    ]
  },
  "id": 1
}
```

**Ответ — отказ на уровне запроса (пустая пачка)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "EmptyOperations",
    "operationResults": null
  },
  "id": 1
}
```

---

## Связанные документы

- [[VehicleService.md]] — доменные методы `findVehicle`, `applyOperations`, `registerVehicle`
- [[VehicleService.Entities.md]] — `Vehicle`, `VehicleOperation`, `VehicleOperationOutcome`
- [[EntityManager.md]] — реестр сущностей; `entityUid` = `EntityItem.uid`
- [[../api/http api.md]] — транспорт JSON-RPC
- [[../ArchBackend/ArchBackend.md]] — VehicleService
- [[../ArchBackend/ServiceAndApiLayers.md]] — Service vs API
