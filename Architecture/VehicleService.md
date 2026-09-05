# VehicleService

Назад: [[Architecture.md]] · [[VehicleService.Entities.md]] · [[VehicleService.HttpMethods.md]]

VehicleService ищет технику на бекенде и применяет пачку операций над ней. Это не реестр сущностей: положение, владение и префаб техники хранит [[EntityManager.md|EntityManager]]. `entityUid` совпадает с [[EntityManager.Entities.md#EntityItem|EntityItem.uid]]; атрибут «последний водитель» живёт здесь, не в EntityManager.

Доменный контракт бекенда: `VehicleService` (`Contracts/VehicleService/`). RPC: [[VehicleService.HttpMethods.md|vehicle@applyOperations]].

## Структура документации

- [[VehicleService.Entities.md]] — `Vehicle`, `VehicleFilter`, `PositionRadiusFilter`, `RegisterVehicle`, `VehicleOperation`, `VehicleOperationOutcome`
- [[VehicleService.HttpMethods.md]] — JSON-RPC `vehicle@applyOperations`

## Методы

### findVehicle

**Сигнатура**
findVehicle(VehicleFilter filter): iterable<Vehicle>

Доменный сервис (`VehicleServiceInterface`):
findVehicle(VehicleFilterDto vehicleFilterDto): iterable<VehicleDto>

**Аргументы**
- `filter: [[VehicleService.Entities.md#VehicleFilter|VehicleFilter]]` — условия выборки. Поля опциональны; заданные комбинируются как AND.

**Описание**
Бекенд возвращает технику, которая удовлетворяет фильтру.

1. Если задан `byLasterDriverCharacterUid` — в выборку попадает техника, у которой последний персонаж на месте водителя равен этому uid.
2. Если задан `positionRadiusFilter` — в выборку попадает техника, расстояние от позиции которой до центра окружности **строго меньше** `radius`.
3. Если заданы оба поля — оба условия должны выполняться одновременно.
4. Пустой iterable — штатный успех: подходящей техники нет.
5. Сервис не бросает доменных исключений по этому методу.

**Результат**
iterable [[VehicleService.Entities.md#Vehicle|Vehicle]] — может быть пустым.

**Ошибки / причины отказа**
- Доменных отказов нет.

**Связано**
[[VehicleService.Entities.md#Vehicle]], [[VehicleService.Entities.md#VehicleFilter]], [[VehicleService.Entities.md#PositionRadiusFilter]], [[EntityManager.Entities.md#EntityItem]]

### applyOperations

**Сигнатура**
applyOperations(VehicleOperation[] operations): VehicleOperationOutcome[]

Доменный сервис (`VehicleServiceInterface`):
applyOperations(list<VehicleOperationDto> vehicleOperations): list<VehicleOperationOutcomeDto>

**Аргументы**
- `operations: [[VehicleService.Entities.md#VehicleOperation|VehicleOperation]][]` — пачка операций в порядке применения.

**Описание**
Бекенд применяет операции по очереди. Результат каждой операции — [[VehicleService.Entities.md#VehicleOperationOutcome|VehicleOperationOutcome]] в том же порядке. `failReason = null` — успех.

1. Транзакция — на одну операцию, не на всю пачку.
2. Если для `entityUid` в этой пачке уже была отказавшая операция, следующие операции с тем же uid получают тот же `failReason` и не применяются.
3. Нет строки `vehicles` по `entityUid` — `VehicleNotFound`. Строка не создаётся.
4. Нет обработчика для `type` — `UnsupportedOperationType`.
5. `ChangeDriver` записывает `lastDriverCharacterUid` = `driverCharacterUid`.
6. Каждая операция логируется (успех и отказ).

**Результат**
Список [[VehicleService.Entities.md#VehicleOperationOutcome|VehicleOperationOutcome]] той же длины, что `operations`.

**Ошибки / причины отказа**
- `VehicleNotFound` — техники с этим `entityUid` нет.
- `UnsupportedOperationType` — тип операции не поддерживается.

**Связано**
[[VehicleService.Entities.md#VehicleOperation]], [[VehicleService.Entities.md#VehicleOperationOutcome]], [[VehicleService.Entities.md#VehicleOperationTypeEnum]], [[VehicleService.Entities.md#ApplyVehicleOperationFailReasonEnum]]

### registerVehicle

**Сигнатура**
registerVehicle(RegisterVehicle registerVehicle): void

Доменный сервис (`VehicleServiceInterface`):
registerVehicle(RegisterVehicleDto addVehicleDto): void

**Аргументы**
- `registerVehicle: [[VehicleService.Entities.md#RegisterVehicle|RegisterVehicle]]` — идентификатор сущности техники в реестре.

**Описание**
Регистрирует технику: создаёт запись с `lastDriverCharacterUid = null`.

1. Если сущности с `entityUid` нет в реестре — `RegisterVehicleEntityNotFoundException`.
2. Иначе добавляется строка техники. Водитель ещё не известен.
3. Повторная регистрация того же uid — инфраструктурная ошибка (уникальный ключ), не доменный отказ.

**Результат**
Нет.

**Ошибки / причины отказа**
- `RegisterVehicleEntityNotFoundException` — сущности с этим `entityUid` нет в реестре.

**Связано**
[[VehicleService.Entities.md#RegisterVehicle]], [[VehicleService.Entities.md#Vehicle]], [[EntityManager.Entities.md#EntityItem]]
