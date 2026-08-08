# HTTP-методы CommandBus (игра → бекенд)

Назад: [[CommandBus.md]] · [[CommandBus.Entities.md]] · [[Commands.md]] · [[../Architecture/Architecture.md]]

Документ описывает JSON-RPC контракты CommandBus / `CommandService` на бекенде:

- опрос пачки ожидающих команд — [[#getPendingCommands]];
- отчёт о выполнении команд — [[#reportCommands]].

Локальные методы игрового сервера (`run`, `reportCommands`): [[CommandBus.md]].  
Каталог типов команд: [[Commands.md]].  
Сущности результата: [[CommandBus.Entities.md]].  
Транспорт и примеры: [[../api/http api.md]].

```text
CommandBus::run (каждые 0.5 с)
  → command@getPendingCommands
  → маршрутизация по type в целевой сервис (например EntityManager.enqueueCommand)
  → сервис → CommandBus::reportCommands
  → command@reportCommands
```

---

## getPendingCommands

**JSON-RPC `method`:** `"command@getPendingCommands"`

**Сигнатура**

```text
getPendingCommands(): GetPendingCommandsResult
```

Доменный сервис (`CommandServiceInterface`):

```text
getPendingCommands(): list<CommandDto>
```

**Аргументы**

Нет. Игровой сервер идентифицируется контекстом API-ключа.

**Описание**

1. Бекенд возвращает пачку команд, ожидающих выполнения в мире.
2. Пустой `commands` — штатный успех (очередь пуста), не ошибка.
3. При выдаче команды переходят в **in-flight**: повторный poll **не** отдаёт их снова, пока нет отчёта через [[#reportCommands]].
4. Возврат в pending после `Fail` или истечения lease — политика бекенда; детали retry-алгоритма в v1 не фиксируются.
5. Сервис не бросает доменных исключений по этому методу; API оборачивает ответ в `GetPendingCommandsResult` со `status = Ok`.
6. На игровом сервере вызывающий — [[CommandBus.md#run|CommandBus::run]]: цикл каждые 0.5 с запрашивает пачку и маршрутизирует команды по `type` (см. [[Commands.md]]).

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
        "type": "SpawnEntity",
        "payload": {
          "entityUid": "ent-backpack-1"
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
reportCommands(CommandReport[] reports): ReportCommandsResult
```

Доменный сервис (`CommandServiceInterface`):

```text
reportCommands(list<CommandReportDto> reports): void
```

**Аргументы**

- `reports: CommandReport[]` — пачка отчётов о выполнении; элемент — [[CommandBus.Entities.md#CommandReport]].

**Описание**

1. Бекенд принимает отчёты и помечает соответствующие команды как завершённые (`Completed`) или невыполненные (`Fail`).
2. Неизвестный `commandUid` или уже завершённая команда — **идемпотентный no-op** для этой записи; остальная пачка обрабатывается.
3. Пустой `reports` — штатный успех.
4. При `status = Fail` в элементе обязателен непустой `failReason`; при `Completed` — `failReason = null`.
5. Сервис не бросает доменных исключений по этому методу при валидной пачке; API оборачивает ответ в `ReportCommandsResult` со `status = Ok`.
6. На уровне API: если `reports` отсутствует или не массив, у элемента нет `commandUid` / `status`, `status` не из enum, или при `Fail` нет `failReason` — `InvalidParams` (request-level Fail).
7. На игровом сервере вызывающий — [[CommandBus.md#reportcommands|CommandBus::reportCommands]] после отчёта целевого сервиса (например EntityManager).

**Результат**

[[CommandBus.Entities.md#ReportCommandsResult]] — при успехе `status = Ok`, `failReason = null`.

**Причины отказа**

Доменных Fail / enum причин нет. На уровне запроса API:

- `InvalidParams` — `reports` отсутствует или не массив; у элемента нет обязательных полей; при `Fail` отсутствует `failReason`.

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
- [[CommandBus.Entities.md]] — `Command`, `CommandReport`, Result DTO
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — целевой сервис для команд сущностей
- [[../api/http api.md]] — транспорт JSON-RPC
