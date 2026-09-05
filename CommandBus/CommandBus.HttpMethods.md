# HTTP-методы CommandBus (игра → бекенд)

Назад: [[CommandBus.md]] · [[CommandBus.Entities.md]] · [[Commands.md]] · [[../Architecture/Architecture.md]]

Документ описывает JSON-RPC контракты CommandBus / `CommandBusService` на бекенде:

- опрос пачки ожидающих блоков команд — [[#getPendingCommandBlocks]];
- пакетный отчёт с опциональным результирующим payload — [[#reportCommands]];
- **deprecated:** плоский опрос команд — [[#getPendingCommands]].

Локальные методы игрового сервера (`run`, `reportCommands`): [[CommandBus.md]].  
Каталог типов команд: [[Commands.md]].  
Сущности результата: [[CommandBus.Entities.md]].  
Транспорт и примеры: [[../api/http api.md]].

```text
CommandBus::run (каждые 0.5 с)
  → command@getPendingCommandBlocks
  → для каждого блока: команды в порядке commands → целевой сервис (например EntityManager.enqueueCommand)
  → сервис → CommandBus::reportCommands
  → command@reportCommands
```

---

## getPendingCommandBlocks

**JSON-RPC `method`:** `"command@getPendingCommandBlocks"`

**Сигнатура**

```text
getPendingCommandBlocks(): GetPendingCommandBlocksResult
```

Доменный сервис (`CommandBusServiceInterface`):

```text
getPendingCommandBlocksAndMarkAsInFlight(): CommandBlockDto[]
```

**Аргументы**

Нет. Игровой сервер идентифицируется контекстом API-ключа.

**Описание**

1. Бекенд возвращает пачку блоков команд текущей игровой сессии ([[../Architecture/GameManager.Entities.md#ServerSession]]), ожидающих выполнения в мире. Блоки других сессий не выдаются и не переводятся в in-flight.
2. Пустой `commandBlocks` — штатный успех (очередь пуста или нет активной сессии), не ошибка.
3. При выдаче блок и его команды переходят в **in-flight**: повторный poll **не** отдаёт их снова, пока нет отчёта через [[#reportCommands]].
4. Команды внутри блока выполняются **строго в порядке** массива `commands`. Игра не переставляет команды внутри блока.
5. Возврат в pending после `Fail` или истечения lease — политика бекенда; детали retry-алгоритма в v1 не фиксируются.
6. При битом payload в хранилище сервис бросает `InvalidArgumentException` (в том числе пустой `prefabList` у `ParsePrefab`). Иначе API оборачивает ответ в `GetPendingCommandBlocksResult` со `status = Ok`.
7. На игровом сервере вызывающий — [[CommandBus.md#run|CommandBus::run]]: цикл каждые 0.5 с запрашивает пачку блоков и маршрутизирует команды по `type` (см. [[Commands.md]]).
8. `serverSessionUid` в result не отдаётся.

**Результат**

[[CommandBus.Entities.md#GetPendingCommandBlocksResult]] — при успехе `status = Ok`, `failReason = null`, `commandBlocks` = массив [[CommandBus.Entities.md#CommandBlock]].

**Причины отказа**

Нет. Доменный Fail / enum причин для этого метода не предусмотрены.

### Примеры

**Запрос**

```json
{
  "jsonrpc": "2.0",
  "method": "command@getPendingCommandBlocks",
  "params": {},
  "id": 1
}
```

**Ответ — успех (пачка блоков)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "commandBlocks": [
      {
        "uid": "blk-001",
        "commands": [
          {
            "uid": "cmd-001",
            "type": "SpawnEntity",
            "payload": {
              "entityUid": "ent-door-001"
            }
          },
          {
            "uid": "cmd-002",
            "type": "SetLockEntity",
            "payload": {
              "entityUid": "ent-door-001",
              "lockEntityUid": "lk01",
              "lockType": "character",
              "isLocked": false
            }
          }
        ]
      },
      {
        "uid": "blk-002",
        "commands": [
          {
            "uid": "cmd-003",
            "type": "ParsePrefab",
            "payload": {
              "prefabList": [
                "Prefabs/Items/example.et"
              ]
            }
          }
        ]
      }
    ]
  },
  "id": 1
}
```

**Ответ — успех (пустая очередь)**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "Ok",
    "failReason": null,
    "commandBlocks": []
  },
  "id": 1
}
```

---

## getPendingCommands

**Deprecated.** Игровой сервер не должен вызывать этот метод. Замена — [[#getPendingCommandBlocks]]. Доменный метод `getPendingCommandsAndMarkAsInFlight` тоже deprecated: отдаёт только одиночные команды без блока (`command_block_uid = null`).

**JSON-RPC `method`:** `"command@getPendingCommands"`

**Сигнатура**

```text
getPendingCommands(): GetPendingCommandsResult
```

Доменный сервис (`CommandBusServiceInterface`):

```text
getPendingCommandsAndMarkAsInFlight(): BaseCommandDto[]
```

**Аргументы**

Нет. Игровой сервер идентифицируется контекстом API-ключа.

**Описание**

1. Бекенд возвращает пачку **одиночных** команд текущей игровой сессии ([[../Architecture/GameManager.Entities.md#ServerSession]]), ожидающих выполнения в мире. Команды внутри блоков и команды других сессий не выдаются и не переводятся в in-flight.
2. Пустой `commands` — штатный успех (очередь пуста или нет активной сессии), не ошибка.
3. При выдаче команды переходят в **in-flight**: повторный poll **не** отдаёт их снова, пока нет отчёта через [[#reportCommands]].
4. Возврат в pending после `Fail` или истечения lease — политика бекенда; детали retry-алгоритма в v1 не фиксируются.
5. При битом payload в хранилище сервис бросает `InvalidArgumentException` (в том числе пустой `prefabList` у `ParsePrefab`). Иначе API оборачивает ответ в `GetPendingCommandsResult` со `status = Ok`.
6. `serverSessionUid` в result не отдаётся.

**Результат**

[[CommandBus.Entities.md#GetPendingCommandsResult]] — при успехе `status = Ok`, `failReason = null`, `commands` = массив [[CommandBus.Entities.md#Command]].

**Причины отказа**

Нет. Доменный Fail / enum причин для этого метода не предусмотрены.

### Примеры

**Запрос**

```json
{
  "jsonrpc": "2.0",
  "method": "command@getPendingCommands",
  "params": {},
  "id": 1
}
```

**Ответ — успех (пачка команд)**

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
      },
      {
        "uid": "cmd-003",
        "type": "ParsePrefab",
        "payload": {
          "prefabList": [
            "Prefabs/Items/example.et"
          ]
        }
      }
    ]
  },
  "id": 1
}
```

**Ответ — успех (пустая очередь)**

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

---

## reportCommands

**JSON-RPC `method`:** `"command@reportCommands"`

**Сигнатура**

```text
reportCommands(BaseReport[] reports): ReportCommandsResult
```

Доменный сервис (`CommandBusServiceInterface`):

```text
reportCommands(BaseReport[] reports): void
```

**Аргументы**

- `reports: BaseReport[]` — пачка отчётов; элемент — [[CommandBus.Entities.md#BaseReport]] с опциональным типизированным `payload`.

**Описание**

1. Бекенд принимает отчёты и помечает соответствующие команды как завершённые (`Completed`) или невыполненные (`Fail`).
2. Каждый отчёт применяется в отдельной транзакции. При ошибке обработка останавливается и исключение пробрасывается; уже применённые отчёты не откатываются.
3. Пустой `reports` — штатный успех.
4. При `status = Fail` в элементе обязателен непустой `failReason`; при `Completed` — `failReason = null`.
5. В каждом элементе обязателен `commandType`. Для `ParsePrefab + Completed` обязателен `payload` типа [[CommandBus.Entities.md#ReportParsePrefabPayload]]. Для `ParsePrefab + Fail` и остальных типов команд результирующий `payload` должен быть `null`.
6. До вызова сервиса API валидирует всю пачку. Если `reports` отсутствует или не массив, у элемента нет `commandUid` / `commandType` / `status`, тип или статус не из enum, при `Fail` нет `failReason` либо payload некорректен — `InvalidParams` (request-level Fail), отчёты не применяются.
7. На игровом сервере вызывающий — [[CommandBus.md#reportcommands|CommandBus::reportCommands]] после отчёта целевого сервиса (например EntityManager).

**Результат**

[[CommandBus.Entities.md#ReportCommandsResult]] — при успехе `status = Ok`, `failReason = null`.

**Причины отказа**

Доменных Fail / enum причин нет. Ошибки состояния команды (команда не найдена, не `InFlight` или `commandType` не соответствует типу команды) приводят к исключению. На уровне запроса API:

- `InvalidParams` — `reports` отсутствует или не массив; у элемента нет обязательных полей; неизвестен `commandType`; при `Fail` отсутствует `failReason`; payload не соответствует типу команды и статусу.

### Примеры

**Запрос**

```json
{
  "jsonrpc": "2.0",
  "method": "command@reportCommands",
  "params": {
    "reports": [
      {
        "commandUid": "cmd-001",
        "commandType": "SpawnEntity",
        "status": "Completed",
        "failReason": null
      },
      {
        "commandUid": "cmd-002",
        "commandType": "SpawnEntity",
        "status": "Fail",
        "failReason": "CannotSpawn"
      },
      {
        "commandUid": "cmd-004",
        "commandType": "ParsePrefab",
        "status": "Completed",
        "failReason": null,
        "payload": {
          "prefabData": [
            {
              "prefabName": "Prefabs/Items/example.et",
              "weight": 1.5,
              "volume": 2.25
            }
          ]
        }
      }
    ]
  },
  "id": 2
}
```

**Ответ — успех**

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

**Ответ — успех (пустая пачка)**

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

**Ответ — отказ на уровне запроса (невалидные params)**

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

---

## См. также

- [[CommandBus.md]] — локальные `run`, `reportCommands`
- [[Commands.md]] — каталог команд и FailReason на стороне игры
- [[CommandBus.Entities.md]] — `CommandBlock`, `Command`, `BaseReport`, payload и Result DTO
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — целевой сервис для команд сущностей
- [[../api/http api.md]] — транспорт JSON-RPC
