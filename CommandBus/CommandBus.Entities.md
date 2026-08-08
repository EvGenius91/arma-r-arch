## Сущности CommandBus

Назад: [[CommandBus.md]] · [[CommandBus.HttpMethods.md]] · [[Commands.md]]

Доменный контракт бекенда: `CommandService` (`Contracts/CommandService/`), DTO `CommandDto` / `CommandReportDto`, enum `CommandTypeEnum` / `CommandExecutionStatusEnum`.

### CommandTypeEnum

**Сигнатура**
```
enum CommandTypeEnum
{
	case SpawnEntity;
}
```

**Назначение**
Тип команды в очереди бекенд → игровой сервер. Соответствует каталогу [[Commands.md]].

**Значения**
- `SpawnEntity` — заспавнить сущность в мире.

**Используется в**
[[#Command]], [[Commands.md]]

### CommandExecutionStatusEnum

**Сигнатура**
```
enum CommandExecutionStatusEnum
{
	case Completed;
	case Fail;
}
```

**Назначение**
Итог выполнения одной команды на игровом сервере (поле `status` в отчёте). Не путать с [[../Architecture/TradeManager.Entities.md#OperationStatusEnum|OperationStatusEnum]] ответа RPC.

**Значения**
- `Completed` — команда успешно применена в мире.
- `Fail` — команда не выполнена; в отчёте обязателен `failReason`.

**Используется в**
[[#CommandReport]]

### Command

**Сигнатура**
```
class Command
{
	string uid;
	CommandTypeEnum type;
	object payload;
}
```

**Назначение**
Команда из очереди бекенда (соответствует `CommandDto`). Элемент списка из [[CommandBus.HttpMethods.md#getPendingCommands]].

**Свойства**
- `uid: string` — идентификатор команды; используется в отчёте как `commandUid`.
- `type: CommandTypeEnum` — тип команды; по нему игровой CommandBus маршрутизирует в целевой сервис.
- `payload: object` — параметры команды; структура зависит от `type` (см. [[#SpawnEntityPayload]]).

**Связано**
[[#CommandTypeEnum]], [[#SpawnEntityPayload]], [[Commands.md]]

**Используется в**
[[#GetPendingCommandsResult]]

### SpawnEntityPayload

**Сигнатура**
```
class SpawnEntityPayload
{
	string entityUid;
}
```

**Назначение**
`payload` команды с `type = SpawnEntity`.

**Свойства**
- `entityUid: string` — идентификатор сущности для спавна в мире.

**Используется в**
[[#Command]], [[Commands.md#SpawnEntity]]

### CommandReport

**Сигнатура**
```
class CommandReport
{
	string commandUid;
	CommandExecutionStatusEnum status;
	string failReason;
}
```

**Назначение**
Отчёт о выполнении одной команды (соответствует элементу пачки в [[CommandBus.HttpMethods.md#reportCommands]]). На игровом сервере целевой сервис передаёт такие записи в `CommandBus::reportCommands`.

**Свойства**
- `commandUid: string` — `Command.uid` из ранее полученной команды.
- `status: CommandExecutionStatusEnum` — `Completed` или `Fail`.
- `failReason: string|null` — при `Completed` всегда `null`; при `Fail` — обязательная текстовая причина (см. [[Commands.md]]).

**Связано**
[[#CommandExecutionStatusEnum]], [[#Command]]

**Используется в**
[[CommandBus.HttpMethods.md#reportCommands]]

### GetPendingCommandsResult

**Сигнатура**
```
class GetPendingCommandsResult
{
	OperationStatusEnum status;
	string failReason;
	Command[] commands;
}
```

**Назначение**
Результат опроса очереди команд ([[CommandBus.HttpMethods.md#getPendingCommands]]). API-обёртка над `list<CommandDto>` сервиса `CommandService`.

**Свойства**
- `status: OperationStatusEnum` — итог операции ([[../Architecture/TradeManager.Entities.md#OperationStatusEnum]]). При успешном вызове сервиса всегда `Ok`. Доменных отказов нет.
- `failReason: string|null` — при `Ok` всегда `null`.
- `commands: Command[]|null` — при `Ok`: пачка команд (может быть пустой); при `Fail` — `null` (зарезервировано).

**Связано**
[[../Architecture/TradeManager.Entities.md#OperationStatusEnum]], [[#Command]]

**Используется в**
[[CommandBus.HttpMethods.md#getPendingCommands]]

### ReportCommandsResult

**Сигнатура**
```
class ReportCommandsResult
{
	OperationStatusEnum status;
	string failReason;
}
```

**Назначение**
Результат приёма пачки отчётов ([[CommandBus.HttpMethods.md#reportCommands]]). API-обёртка над `void` сервиса `CommandService`.

**Свойства**
- `status: OperationStatusEnum` — итог приёма пачки. При успешном вызове сервиса всегда `Ok`. Request-level `Fail` возможен при невалидных params (`InvalidParams`).
- `failReason: string|null` — при `Ok` всегда `null`; при request Fail — `"InvalidParams"`.

**Связано**
[[../Architecture/TradeManager.Entities.md#OperationStatusEnum]]

**Используется в**
[[CommandBus.HttpMethods.md#reportCommands]]
