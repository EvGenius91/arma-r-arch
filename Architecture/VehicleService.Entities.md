## Сущности VehicleService

Назад: [[VehicleService.md]]

Доменный контракт бекенда: `VehicleService` (`Contracts/VehicleService/`), DTO `VehicleDto` / `VehicleFilterDto` / `RegisterVehicleDto` / `VehicleOperationDto` / `VehicleOperationOutcomeDto`. `PositionRadiusFilterDto` — общий (`Contracts/Common/Dto/`).

### Vehicle

**Сигнатура**
```
class Vehicle
{
	string entityUid;
	string|null lastDriverCharacterUid;
}
```

**Назначение**
Техника в выборке [[VehicleService.md#findVehicle]]. Соответствует `VehicleDto`. Не заменяет [[EntityManager.Entities.md#EntityItem|EntityItem]]: здесь только идентификатор сущности и последний водитель.

**Свойства**
- `entityUid: string` - идентификатор сущности техники; совпадает с [[EntityManager.Entities.md#EntityItem|EntityItem.uid]].
- `lastDriverCharacterUid: string|null` - uid персонажа, который последним был на месте водителя; `null`, если технику ещё не садили (только что заспавнена).

**Связано**
[[EntityManager.Entities.md#EntityItem]]

**Используется в**
[[VehicleService.md#findVehicle]], [[VehicleService.md#applyOperations]], [[VehicleService.md#registerVehicle]]

### RegisterVehicle

**Сигнатура**
```
class RegisterVehicle
{
	string entityUid;
}
```

**Назначение**
Данные для регистрации техники ([[VehicleService.md#registerVehicle]]). Соответствует `RegisterVehicleDto`.

**Свойства**
- `entityUid: string` - идентификатор сущности техники; должен существовать в реестре ([[EntityManager.Entities.md#EntityItem]]).

**Связано**
[[#Vehicle]], [[EntityManager.Entities.md#EntityItem]]

**Используется в**
[[VehicleService.md#registerVehicle]]

### VehicleFilter

**Сигнатура**
```
class VehicleFilter
{
	string byLasterDriverCharacterUid;
	PositionRadiusFilter positionRadiusFilter;
}
```

**Назначение**
Условия поиска техники ([[VehicleService.md#findVehicle]]). Соответствует `VehicleFilterDto`. Оба поля опциональны; заданные комбинируются как AND.

**Свойства**
- `byLasterDriverCharacterUid: string|null` - идентификатор персонажа, который последним был на месте водителя; `null` — не фильтровать по водителю. Имя поля — как в контракте (`VehicleFilterDto`).
- `positionRadiusFilter: PositionRadiusFilter|null` - фильтр по расстоянию от точки в мире; `null` — не фильтровать по позиции.

**Связано**
[[#PositionRadiusFilter]]

**Используется в**
[[VehicleService.md#findVehicle]]

### PositionRadiusFilter

**Сигнатура**
```
class PositionRadiusFilter
{
	Position position;
	float radius;
}
```

**Назначение**
Фильтр по расстоянию от точки в мире. Соответствует `PositionRadiusFilterDto` (`Contracts/Common/Dto/`). Сущность попадает в выборку, если расстояние от её позиции до `position` **строго меньше** `radius`.

Общий DTO: тем же типом пользуется `EntityRegistryService` (`EntityFilterDto.positionRadiusList`).

**Свойства**
- `position: Position` - центр окружности в мире ([[EntityManager.Entities.md#Position]]).
- `radius: float` - радиус выборки.

**Связано**
[[EntityManager.Entities.md#Position]]

**Используется в**
[[#VehicleFilter]]

### VehicleOperation

**Сигнатура**
```
class VehicleOperation
{
	string entityUid;
	VehicleOperationTypeEnum type;
	string driverCharacterUid;
}
```

**Назначение**
Одна операция над техникой в пачке [[VehicleService.md#applyOperations]]. Соответствует `VehicleOperationDto`.

**Свойства**
- `entityUid: string` - идентификатор сущности техники; совпадает с [[#Vehicle|Vehicle.entityUid]].
- `type: VehicleOperationTypeEnum` - тип операции.
- `driverCharacterUid: string` - uid персонажа на месте водителя. Для `ChangeDriver` становится новым `lastDriverCharacterUid`.

**Связано**
[[#VehicleOperationTypeEnum]], [[#Vehicle]]

**Используется в**
[[VehicleService.md#applyOperations]], [[VehicleService.HttpMethods.md#applyOperations]]

### VehicleOperationOutcome

**Сигнатура**
```
class VehicleOperationOutcome
{
	string entityUid;
	VehicleOperationTypeEnum type;
	ApplyVehicleOperationFailReasonEnum failReason;
}
```

**Назначение**
Результат одной операции из [[VehicleService.md#applyOperations]]. Соответствует `VehicleOperationOutcomeDto`. `failReason = null` — успех.

**Свойства**
- `entityUid: string` - идентификатор сущности из операции.
- `type: VehicleOperationTypeEnum` - тип операции.
- `failReason: ApplyVehicleOperationFailReasonEnum|null` - причина отказа; `null` при успехе.

**Связано**
[[#ApplyVehicleOperationFailReasonEnum]], [[#VehicleOperationTypeEnum]]

**Используется в**
[[VehicleService.md#applyOperations]], [[VehicleService.HttpMethods.md#applyOperations]]

### VehicleOperationTypeEnum

**Сигнатура**
```
enum VehicleOperationTypeEnum
{
	changeDriver;
}
```

**Назначение**
Тип операции над техникой. Соответствует `VehicleOperationTypeEnum`.

**Значения**
- `changeDriver` — персонаж занял место водителя; бекенд записывает `lastDriverCharacterUid`.

**Используется в**
[[#VehicleOperation]], [[#VehicleOperationOutcome]]

### ApplyVehicleOperationFailReasonEnum

**Сигнатура**
```
enum ApplyVehicleOperationFailReasonEnum
{
	VehicleNotFound;
	UnsupportedOperationType;
}
```

**Назначение**
Причина отказа одной операции [[VehicleService.md#applyOperations]]. Соответствует `ApplyVehicleOperationFailReasonEnum`.

**Значения**
- `VehicleNotFound` — строки техники с этим `entityUid` нет; запись не создаётся.
- `UnsupportedOperationType` — для `type` нет обработчика.

**Используется в**
[[#VehicleOperationOutcome]]
