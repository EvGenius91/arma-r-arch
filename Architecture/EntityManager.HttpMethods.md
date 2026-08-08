# HTTP-методы EntityManager (игра → бекенд)

Назад: [[EntityManager.md]] · [[EntityManager.Operations.md]] · [[EntityManager.Entities.md]] · [[Architecture.md]]

Документ описывает JSON-RPC контракты EntityManager / `EntityRegistryService` на бекенде:

- чтение инвентаря персонажа — [[#getInventoryByCharacterUid]];
- чтение сущностей по списку uid — [[#findEntitiesByUidList]];
- отправка пачки операций расположения при flush — [[#applyOperations]] (таймер ~1 с или barrier; вызывается **только из EntityManager**; инвентарь / мир / UI / CharacterStateManager этот RPC не вызывают).

Очередь, lock на in-flight, barrier: [[EntityManager.Operations.md]].  
`resetGeneration` и `StaleAfterReset`: [[EntityManager.DupeAnalyzer.md]].  
Сущности результата: [[EntityManager.Entities.md]].  
Транспорт и примеры: [[../api/http api.md]].

```text
Game EM flush → Lock uids → entity@applyOperations(operations[]) → per-op Ok/Fail → Unlock / rollback
```

---

## getInventoryByCharacterUid

**JSON-RPC `method`:** `"entity@getInventoryByCharacterUid"`

**Сигнатура**

```text
getInventoryByCharacterUid(string characterUid): GetInventoryByCharacterUidResult
```

Доменный сервис (`EntityRegistryServiceInterface`):

```text
getInventoryByCharacterUid(string characterUid): list<EntityItemDto>
```

**Аргументы**

- `characterUid: string` — персонаж, чей инвентарь / носимый набор запрашивается.

**Описание**

1. По `characterUid` бекенд возвращает **плоский** список сущностей из реестра, принадлежащих инвентарю персонажа (включая контейнеры и вложенные предметы).
2. Это не агрегат Trade (`getInventoryForSell`) и не дерево: вложенность клиент восстанавливает по `parentContainerUid` / `inventorySlotUid`.
3. Пустой список — штатный успех (у персонажа нет сущностей в инвентаре), не ошибка.
4. Сервис не бросает доменных исключений по этому методу; API оборачивает ответ в `GetInventoryByCharacterUidResult` со `status = Ok`.
5. На игровом сервере при спавне / респавне вызывающий — [[EntityManager.md#loadInventory|EntityManager.loadInventory]]: он запрашивает этот RPC и материализует сущности на новом персонаже.

**Результат**

[[EntityManager.Entities.md#GetInventoryByCharacterUidResult]] — при успехе `status = Ok`, `failReason = null`, `entities` = массив [[EntityManager.Entities.md#EntityItem]].

**Причины отказа**

Нет. Доменный Fail / enum причин для этого метода не предусмотрены.

### Примеры

**Запрос**

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

**Ответ — успех (контейнер и предмет внутри)**

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

**Ответ — успех (пустой инвентарь)**

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

---

## findEntitiesByUidList

**JSON-RPC `method`:** `"entity@findEntitiesByUidList"`

**Сигнатура**

```text
findEntitiesByUidList(string[] uidList): FindEntitiesByUidListResult
```

Доменный сервис (`EntityRegistryServiceInterface`):

```text
findEntitiesByUidList(list<string> uidList): list<EntityItemDto>
```

**Аргументы**

- `uidList: string[]` — список идентификаторов сущностей для выборки.

**Описание**

1. Бекенд возвращает сущности из реестра по переданным uid.
2. Отсутствующие uid **пропускаются** — это не ошибка и не `Fail`.
3. Порядок элементов в `entities` соответствует порядку запрошенных uid (только найденные).
4. Пустой `uidList` → пустой `entities` (штатный успех).
5. Скрытые сущности в результат не попадают.
6. Сервис не бросает доменных исключений по этому методу; API оборачивает ответ в `FindEntitiesByUidListResult` со `status = Ok`.
7. На уровне API: если `uidList` отсутствует или не массив — `InvalidParams` (request-level Fail).

**Результат**

[[EntityManager.Entities.md#FindEntitiesByUidListResult]] — при успехе `status = Ok`, `failReason = null`, `entities` = массив [[EntityManager.Entities.md#EntityItem]].

**Причины отказа**

Доменных Fail / enum причин нет. На уровне запроса API:

- `InvalidParams` — `uidList` отсутствует или не массив.

### Примеры

**Запрос**

```json
{
  "jsonrpc": "2.0",
  "method": "entity@findEntitiesByUidList",
  "params": {
    "uidList": ["ent-can-001", "ent-missing", "ent-backpack-1"]
  },
  "id": 2
}
```

**Ответ — успех (найденные; отсутствующий uid пропущен, порядок сохранён)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "entities": [
      {
        "uid": "ent-can-001",
        "prefabName": "Food_Can",
        "isContainer": false,
        "position": null,
        "parentContainerUid": "ent-backpack-1",
        "inventorySlotUid": null,
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      },
      {
        "uid": "ent-backpack-1",
        "prefabName": "Backpack_01",
        "isContainer": true,
        "position": null,
        "parentContainerUid": null,
        "inventorySlotUid": "slot-backpack",
        "ownerCharacterUid": "char-42",
        "storageType": "SCR_CharacterInventoryStorageComponent"
      }
    ]
  },
  "id": 2
}
```

**Ответ — успех (пустой список uid)**

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

**Ответ — отказ на уровне запроса (невалидные params)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Fail",
    "failReason": "InvalidParams",
    "entities": null
  },
  "id": 2
}
```

---

## applyOperations

**JSON-RPC `method`:** `"entity@applyOperations"`

**Сигнатура**

```text
applyOperations(operations[]): ApplyEntityOperationsResult
```

**Аргументы**

- `operations: EntityOperation[]` — непустой упорядоченный список операций. Порядок в массиве = порядок применения на бекенде.

### Элемент `EntityOperation`

| Поле | Тип | Описание |
|------|-----|----------|
| `entityUid` | string | Идентификатор сущности |
| `type` | string | Тип операции (см. каталог ниже) |
| `resetGeneration` | int | Поколение сущности на момент отправки |
| `characterUid` | string | Персонаж-инициатор (как в `enqueue*`) |
| `payload` | object | Поля, зависящие от `type` |

### Каталог `type`

| `type` | Смысл |
|--------|--------|
| `PickUpEntity` | из мира → контейнер / слот |
| `DropEntity` | из инвентаря / слота → мир |
| `MoveEntity` | между слотами / контейнерами |
| `EquipItem` | надеть |
| `UnequipItem` | снять |
| `SwapEquipment` | обмен слотов |
| `TransferEntity` | передача другому персонажу |
| `SplitStack` | разделение стека (если есть в проекте) |
| `MergeStack` | слияние стеков (если есть в проекте) |
| `DestroyEntity` | уничтожение / снятие с реестра (поля `payload` уточняются отдельно) |

### `payload` по типам

Общее обязательное поле `payload` для операций расположения (в т.ч. PickUp / Move / Drop):

| Поле | Тип | Описание |
|------|-----|----------|
| `storageType` | string | Тип хранилища ([[EntityManager.Entities.md#StorageTypeEnum]]): `SCR_CharacterInventoryStorageComponent`, `SCR_WeaponAttachmentsStorageComponent`, `EquipedWeaponStorageComponent`, `SCR_UniversalInventoryStorageComponent` |

**`PickUpEntity` / `MoveEntity`**

| Поле                 | Тип            | Описание                                                                 |
| -------------------- | -------------- | ------------------------------------------------------------------------ |
| `targetContainerUid` | string \| null | Контейнер назначения; `null`, если цель — корневой слот экипировки персонажа (рюкзак, штаны, куртка и т.п.) |
| `slot`               | string \| null | Слот назначения; обязателен, когда `targetContainerUid` = `null`         |
| `storageType`        | string         | Тип хранилища назначения (обязателен)                                    |

**`DropEntity`**

| Поле | Тип | Описание |
|------|-----|----------|
| `position` | object | Координаты дропа: `{ x: number, y: number, z: number }` |
| `storageType` | string | Тип хранилища (обязателен) |

**`TransferEntity`**

| Поле | Тип | Описание |
|------|-----|----------|
| `toCharacterUid` | string | Получатель |
| `targetContainerUid` | string \| null | Контейнер назначения у получателя, если нужен |
| `slot` | string \| null | Слот назначения, если нужен |
| `storageType` | string | Тип хранилища назначения (обязателен) |

**`EquipItem` / `UnequipItem` / `SwapEquipment` / `SplitStack` / `MergeStack`**

Поля `payload` — по доменным правилам экипировки и стеков; должны однозначно задавать целевое расположение. Каталог смыслов: [[EntityManager.md#каталог-операций-расположения-игра--бекенд]].

**`DestroyEntity`**

Контракт `payload` уточняется отдельно (минимум достаточно `entityUid` + `resetGeneration` на уровне ops).

### Описание

1. EntityManager при flush собирает pending ops в пачку с сохранением порядка и вызывает этот метод.
2. Бекенд применяет операции **по порядку** в `operations`.
3. В рамках одного `entityUid` — строго последовательно. Если ранняя ops по uid дала `Fail`, последующие ops того же uid в этой пачке получают `Fail` без применения.
4. Пачка **не** атомарна по всем uid: отказ по uid A не откатывает уже применённые ops по uid B.
5. Второй одновременный успешный take одной сущности → `AlreadyTaken`.
6. Если `op.resetGeneration` не совпадает с текущим поколением сущности на бекенде → `StaleAfterReset`, мир по этой ops не двигать ([[EntityManager.DupeAnalyzer.md]]).
7. Этот RPC обновляет истину владения / контейнера / позиции на бекенде. Мутации игрового мира с бекенда (спавн, `removeEntity` и т.п.) — через CommandBus ([[BackendGameMutation.md]]), не вместо этого метода.

### Результат (`ApplyEntityOperationsResult`)

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | string | `Ok` — пачка обработана (в т.ч. с частичными Fail по ops); `Fail` — отказ на уровне запроса |
| `failReason` | string \| null | При request-level Fail — причина; иначе `null` |
| `operationResults` | EntityOperationResult[] \| null | При `Ok`: массив той же длины и порядка, что `operations`; при request Fail — `null` |

Элемент `operationResults[]` (`EntityOperationResult`):

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | string | `Ok` или `Fail` |
| `failReason` | string \| null | При Fail — причина; при Ok — `null` |

Индексация **1:1** с входным `operations[]` (без дублирования `entityUid` в результате — идентификация по индексу).

Игровой сервер по индексам откатывает расположение в мире только для упавших ops и снимает lock затронутых uid ([[EntityManager.Operations.md]]).

### Причины отказа на уровне запроса (`failReason`)

- `EmptyOperations` — массив `operations` пуст.
- `InvalidParams` — невалидная структура params / элемента ops до доменной обработки.

### Причины отказа на уровне операции (`operationResults[].failReason`)

- `EntityNotFound` — сущность не найдена на бекенде.
- `StaleAfterReset` — устаревшее `resetGeneration` после hard-reset.
- `ContainerNotFound` — целевой контейнер не найден.
- `InvalidLocation` — недопустимое целевое расположение / слот.
- `AlreadyTaken` — сущность уже забрана / конфликт владения (второй take).
- `InvalidOperation` — прочая доменная недопустимость операции.

---

## Примеры

### Запрос — Drop затем PickUp одной банки

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

### Ответ — обе ops Ok

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

### Ответ — частичный успех (вторая ops AlreadyTaken)

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

### Ответ — StaleAfterReset

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

### Ответ — отказ на уровне запроса (пустая пачка)

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

---

## Связанные документы

- [[EntityManager.md]] — реестр и `enqueue*`
- [[EntityManager.Entities.md]] — `EntityItem`, `GetInventoryByCharacterUidResult`, `FindEntitiesByUidListResult`
- [[EntityManager.Operations.md]] — очередь, flush, in-flight
- [[EntityManager.DupeAnalyzer.md]] — `resetGeneration`, hard-reset
- [[EntityLockRegistry.md]] — Lock на время in-flight
- [[BackendGameMutation.md]] — бекенд → CommandBus → мир
- [[../api/http api.md]] — транспорт JSON-RPC
- [[../ArchBackend/ArchBackend.md]] — EntityRegistryService
- [[../ArchBackend/ServiceAndApiLayers.md]] — Service vs API
