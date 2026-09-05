## Сущности CommandBus

Назад: [[CommandBus.md]] · [[CommandBus.HttpMethods.md]] · [[Commands.md]]

Доменный контракт бекенда: `CommandBusService` (`Contracts/CommandBus/`), DTO `CommandBlockDto` / `CommandDto` / `BaseCommandDto` / `BaseReport` / `AddCommandBlockDto`, enum `CommandTypeEnum` / `CommandStatusEnum`.

### CommandTypeEnum

**Сигнатура**
```
enum CommandTypeEnum
{
	case SpawnEntity;
	case DeleteEntity;
	case ParsePrefab;
	case SetLockEntity;
	case ChangeLockStatus;
	case AddSafeZone;
	case SetEntityOwner;
	case SetAccessByType;
}
```

**Назначение**
Тип команды в очереди бекенд → игровой сервер. Соответствует каталогу [[Commands.md]].

**Значения**
- `SpawnEntity` — заспавнить сущность в мире.
- `DeleteEntity` — удалить сущность из мира.
- `ParsePrefab` — получить характеристики списка префабов.
- `SetLockEntity` — повесить замок на сущность.
- `ChangeLockStatus` — сменить статус замка (открыт / закрыт).
- `AddSafeZone` — записать safe-zone в игровой `TSM_SafeZoneService`. JSON-значение `type` — `AddSafeZones`.
- `SetEntityOwner` — задать `TSM_EntityProps.ownerUidList` у сущности.
- `SetAccessByType` — записать одно разрешение в карту игрового `TSM_AccessService`.

**Используется в**
[[#Command]], [[Commands.md]]

### CommandStatusEnum

**Сигнатура**
```
enum CommandStatusEnum
{
	case Pending;
	case InFlight;
	case Completed;
	case Fail;
}
```

**Назначение**
Состояние команды и блока в очереди. Один enum для обоих. В JSON-RPC-отчёте используются только итоговые значения `Completed` и `Fail`. Не путать с [[../Architecture/TradeManager.Entities.md#OperationStatusEnum|OperationStatusEnum]] ответа RPC.

**Значения**
- `Pending` — команда или блок ожидает выдачи игровому серверу.
- `InFlight` — команда или блок выданы игровому серверу и ожидают отчёта.
- `Completed` — команда успешно применена в мире.
- `Fail` — команда не выполнена; в отчёте обязателен `failReason`.

**Используется в**
[[#Command]], [[#CommandBlock]], [[#BaseReport]]

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
Команда из очереди бекенда. В HTTP и внутри блока соответствует `CommandDto` (поле типа — `type`; в доменном DTO — `commandType`). Элемент [[#CommandBlock]] из [[CommandBus.HttpMethods.md#getPendingCommandBlocks]]. Также элемент deprecated-списка из [[CommandBus.HttpMethods.md#getPendingCommands]] (`BaseCommandDto`).

**Свойства**
- `uid: string` — идентификатор команды; используется в отчёте как `commandUid`.
- `type: CommandTypeEnum` — тип команды; по нему игровой CommandBus маршрутизирует в целевой сервис.
- `payload: object` — параметры команды; структура зависит от `type` (см. [[#SpawnEntityPayload]], [[#DeleteEntityPayload]], [[#ParsePrefabPayload]], [[#SetLockEntityPayload]], [[#ChangeLockStatusPayload]], [[#AddSafeZonePayload]], [[#SetEntityOwnerPayload]], [[#SetAccessByTypePayload]]).

**Связано**
[[#CommandTypeEnum]], [[#SpawnEntityPayload]], [[#DeleteEntityPayload]], [[#ParsePrefabPayload]], [[#SetLockEntityPayload]], [[#ChangeLockStatusPayload]], [[#AddSafeZonePayload]], [[#SetEntityOwnerPayload]], [[#SetAccessByTypePayload]], [[Commands.md]]

**Используется в**
[[#CommandBlock]], [[#GetPendingCommandBlocksResult]], [[#GetPendingCommandsResult]]

### CommandBlock

**Сигнатура**
```
class CommandBlock
{
	string uid;
	Command[] commands;
}
```

**Назначение**
Блок команд для игрового сервера (соответствует `CommandBlockDto`). Команды внутри блока выполняются **строго в порядке** массива `commands`. Элемент списка из [[CommandBus.HttpMethods.md#getPendingCommandBlocks]].

**Свойства**
- `uid: string` — идентификатор блока.
- `commands: Command[]` — непустой список команд в порядке выполнения.

**Связано**
[[#Command]], [[#CommandStatusEnum]]

**Используется в**
[[#GetPendingCommandBlocksResult]], [[CommandBus.HttpMethods.md#getPendingCommandBlocks]]

### AddCommand

**Сигнатура**
```
class AddCommand
{
	CommandTypeEnum commandType;
	object payload;
}
```

**Назначение**
Элемент постановки команды в блок. Соответствует `AddCommandDto`. Не входит в HTTP игры → бекенд.

**Свойства**
- `commandType: CommandTypeEnum` — тип команды.
- `payload: object` — параметры команды; структура зависит от `commandType`.

**Используется в**
[[#AddCommandBlock]]

### AddCommandBlock

**Сигнатура**
```
class AddCommandBlock
{
	AddCommand[] addCommand;
}
```

**Назначение**
Постановка блока в очередь (`addCommandBlock`, пакетно — `addCommandBlockBatch`). Соответствует `AddCommandBlockDto`. Список `addCommand` непустой. Не входит в HTTP игры → бекенд.

**Свойства**
- `addCommand: AddCommand[]` — команды блока в порядке выполнения.

**Используется в**
`CommandBusServiceInterface.addCommandBlock`, `CommandBusServiceInterface.addCommandBlockBatch`

### AddCommandBlockBatch

**Сигнатура**
```
addCommandBlockBatch(AddCommandBlock[] addCommandBlockDtoList): void
```

**Назначение**
Пакетная постановка блоков в очередь (`CommandBusServiceInterface.addCommandBlockBatch`). Пустой список допустим (no-op, активная сессия не нужна). Каждый элемент — непустой [[#AddCommandBlock]]. Команды внутри блока выполняются в порядке `addCommand`. Порядок блоков между собой не сильнее сортировки выдачи по `created_at` и `uuid`. Не входит в HTTP игры → бекенд.

**Связано**
[[#AddCommandBlock]]

**Используется в**
`CommandBusServiceInterface.addCommandBlockBatch`

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

### DeleteEntityPayload

**Сигнатура**
```
class DeleteEntityPayload
{
	string entityUid;
}
```

**Назначение**
`payload` команды с `type = DeleteEntity`.

**Свойства**
- `entityUid: string` — идентификатор сущности для удаления из мира.

**Используется в**
[[#Command]], [[Commands.md#DeleteEntity]]

### ParsePrefabPayload

**Сигнатура**
```
class ParsePrefabPayload
{
	string[] prefabList;
}
```

**Назначение**
`payload` команды с `type = ParsePrefab`.

**Свойства**
- `prefabList: string[]` — непустой список имён префабов, характеристики которых нужно определить.

**Используется в**
[[#Command]], [[Commands.md#ParsePrefab]]

### SetLockEntityPayload

**Сигнатура**
```
class SetLockEntityPayload
{
	string entityUid;
	string lockEntityUid;
	LockTypeEnum lockType;
	bool isLocked;
}
```

**Назначение**
`payload` команды с `type = SetLockEntity`.

**Свойства**
- `entityUid: string` — сущность, на которую ставится замок.
- `lockEntityUid: string` — идентификатор замка.
- `lockType: LockTypeEnum` — тип замка ([[../Architecture/LockManager.Entities.md#LockTypeEnum]]): `character`, `pinCode`, `keyEntity`.
- `isLocked: bool` — закрыт ли замок.

**Используется в**
[[#Command]], [[Commands.md#SetLockEntity]]

### ChangeLockStatusPayload

**Сигнатура**
```
class ChangeLockStatusPayload
{
	string lockUid;
	bool isLocked;
}
```

**Назначение**
`payload` команды с `type = ChangeLockStatus`.

**Свойства**
- `lockUid: string` — идентификатор замка.
- `isLocked: bool` — закрыт (`true`) или открыт (`false`).

**Используется в**
[[#Command]], [[Commands.md#ChangeLockStatus]]

### AddSafeZonePayload

**Сигнатура**
```
class AddSafeZonePayload
{
	string safeZoneUid;
	string safeZoneName;
	Position centerPosition;
	float radius;
}
```

**Назначение**
`payload` команды с `type = AddSafeZone` (JSON `type` — `AddSafeZones`).

**Свойства**
- `safeZoneUid: string` — идентификатор safe-zone.
- `safeZoneName: string` — отображаемое имя safe-zone.
- `centerPosition: [[../Architecture/EntityManager.Entities.md#Position|Position]]` — центр зоны в мире.
- `radius: float` — радиус зоны.

**Используется в**
[[#Command]], [[Commands.md#AddSafeZone]]

### SetEntityOwnerPayload

**Сигнатура**
```
class SetEntityOwnerPayload
{
	string entityUid;
	string[]|null ownerUidList;
}
```

**Назначение**
`payload` команды с `type = SetEntityOwner`. Передаёт сразу всех владельцев (полная замена списка).

**Свойства**
- `entityUid: string` — сущность, у которой задаются владельцы.
- `ownerUidList: string[]|null` — uid персонажей-владельцев (`TSM_EntityProps.ownerUidList`); `null` — игра не проверяет владельца.

**Используется в**
[[#Command]], [[Commands.md#SetEntityOwner]]

### SetAccessByTypePayload

**Сигнатура**
```
class SetAccessByTypePayload
{
	string entityUid;
	AccessPermissionEnum permissionType;
	string[]|null characterUidList;
}
```

**Назначение**
`payload` команды с `type = SetAccessByType`. Заменяет грант только для пары `entityUid` + `permissionType`.

**Свойства**
- `entityUid: string` — сущность, у которой обновляется одно разрешение.
- `permissionType: [[../Architecture/AccessService.Entities.md#AccessPermissionEnum|AccessPermissionEnum]]` — тип права; JSON-строка `GetIn` \| `AccessToInventory` \| `AccessToOpenCloseLock`.
- `characterUidList: string[]|null` — uid персонажей с правом. `null` — разрешено всем; `[]` — никому; непустой список — только эти персонажи. `null` и пустой массив не эквивалентны.

**Используется в**
[[#Command]], [[Commands.md#SetAccessByType]]

### PrefabDataDto

**Сигнатура**
```
class PrefabDataDto
{
	string prefabName;
	float weight;
	float volume;
}
```

**Назначение**
Характеристики одного префаба, вычисленные игровым сервером.

**Свойства**
- `prefabName: string` — имя префаба из команды.
- `weight: float` — вес сущности.
- `volume: float` — объём сущности.

**Используется в**
[[#ReportParsePrefabPayload]]

### ReportParsePrefabPayload

**Сигнатура**
```
class ReportParsePrefabPayload
{
	PrefabDataDto[] prefabData;
}
```

**Назначение**
Результирующий payload отчёта команды `ParsePrefab`.

**Свойства**
- `prefabData: PrefabDataDto[]` — список характеристик префабов; может быть пустым.

**Используется в**
[[#BaseReport]], [[CommandBus.HttpMethods.md#reportCommands]]

### BaseReport

**Сигнатура**
```
readonly class BaseReport
{
	string commandUid;
	CommandTypeEnum commandType;
	CommandStatusEnum status;
	string|null failReason;
	object|null payload;
}
```

**Назначение**
Отчёт о выполнении одной команды. На уровне доменного DTO поле статуса называется `commandStatus`; в JSON-RPC передаётся как `status`.

Все отчёты передаются пачкой через [[CommandBus.HttpMethods.md#reportCommands]]. Каждый элемент может содержать типизированный результирующий payload.

**Свойства**
- `commandUid: string` — `Command.uid` из ранее полученной команды.
- `commandType: CommandTypeEnum` — тип ранее полученной команды.
- `status: CommandStatusEnum` — только `Completed` или `Fail`.
- `failReason: string|null` — при `Completed` всегда `null`; при `Fail` — обязательная непустая текстовая причина (см. [[Commands.md]]).
- `payload: object|null` — результат выполнения команды. Для `ParsePrefab + Completed` обязателен [[#ReportParsePrefabPayload]]; для `ParsePrefab + Fail` и всех остальных типов команд должен быть `null`.

**Связано**
[[#CommandStatusEnum]], [[#Command]], [[#ReportParsePrefabPayload]]

**Используется в**
[[CommandBus.HttpMethods.md#reportCommands]]

### GetPendingCommandBlocksResult

**Сигнатура**
```
class GetPendingCommandBlocksResult
{
	OperationStatusEnum status;
	string failReason;
	CommandBlock[] commandBlocks;
}
```

**Назначение**
Результат опроса очереди блоков ([[CommandBus.HttpMethods.md#getPendingCommandBlocks]]). API-обёртка над `CommandBlockDto[]` сервиса `CommandBusService`.

**Свойства**
- `status: OperationStatusEnum` — итог операции ([[../Architecture/TradeManager.Entities.md#OperationStatusEnum]]). При успешном вызове сервиса всегда `Ok`. Доменных отказов нет.
- `failReason: string|null` — при `Ok` всегда `null`.
- `commandBlocks: CommandBlock[]|null` — при `Ok`: пачка блоков (может быть пустой); при `Fail` — `null` (зарезервировано).

**Связано**
[[../Architecture/TradeManager.Entities.md#OperationStatusEnum]], [[#CommandBlock]]

**Используется в**
[[CommandBus.HttpMethods.md#getPendingCommandBlocks]]

### GetPendingCommandsResult

**Deprecated.** Замена — [[#GetPendingCommandBlocksResult]].

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
Результат deprecated-опроса одиночных команд ([[CommandBus.HttpMethods.md#getPendingCommands]]). API-обёртка над `BaseCommandDto[]` сервиса `CommandBusService`.

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
Результат приёма отчётов ([[CommandBus.HttpMethods.md#reportCommands]]). API-обёртка над `void` сервиса `CommandBusService`.

**Свойства**
- `status: OperationStatusEnum` — итог приёма пачки отчётов. При успешном вызове сервиса всегда `Ok`. Request-level `Fail` возможен при невалидных params (`InvalidParams`).
- `failReason: string|null` — при `Ok` всегда `null`; при request Fail — `"InvalidParams"`.

**Связано**
[[../Architecture/TradeManager.Entities.md#OperationStatusEnum]]

**Используется в**
[[CommandBus.HttpMethods.md#reportCommands]]
