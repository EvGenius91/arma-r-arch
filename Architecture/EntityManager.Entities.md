## Сущности EntityManager

Назад: [[EntityManager.md]] · [[EntityManager.HttpMethods.md]]

Доменный контракт бекенда: `EntityRegistryService` (`Contracts/EntityRegistryService/`), DTO `EntityItemDto` / `EntityHitZoneDto` / `EntityParamsDto` / `PositionDto` / `AngleDto`, enum `StorageTypeEnum`.

### StorageTypeEnum

**Сигнатура**
```
enum StorageTypeEnum
{
	SCR_CharacterInventoryStorageComponent;
	SCR_WeaponAttachmentsStorageComponent;
	EquipedWeaponStorageComponent;
	SCR_UniversalInventoryStorageComponent;
	SCR_EquipmentStorageComponent;
}
```

**Назначение**
Тип хранилища сущности (компонент инвентаря / экипировки в мире). Соответствует `StorageTypeEnum` на бекенде. Обязателен в `EntityItem` и в `payload` операций расположения (`applyOperations`).

**Значения**
- `SCR_CharacterInventoryStorageComponent` — хранилище инвентаря персонажа.
- `SCR_WeaponAttachmentsStorageComponent` — хранилище вложений оружия.
- `EquipedWeaponStorageComponent` — хранилище экипированного оружия.
- `SCR_UniversalInventoryStorageComponent` — универсальное хранилище инвентаря.

**Используется в**
[[#EntityItem]], [[EntityManager.HttpMethods.md#applyOperations]]

### Position

**Сигнатура**
```
class Position
{
	float x;
	float y;
	float z;
}
```

**Назначение**
Координаты сущности в игровом мире (соответствует `PositionDto` на бекенде).

**Свойства**
- `x: float` - координата X.
- `y: float` - координата Y.
- `z: float` - координата Z.

**Используется в**
[[#EntityItem]], [[EntityManager.HttpMethods.md#entitytransformchanged]]

### Angle

**Сигнатура**
```
class Angle
{
	float x;
	float y;
	float z;
}
```

**Назначение**
Угол сущности в игровом мире (соответствует `AngleDto` на бекенде).

**Свойства**
- `x: float` - угол вокруг оси X.
- `y: float` - угол вокруг оси Y.
- `z: float` - угол вокруг оси Z.

**Используется в**
[[#EntityItem]], [[EntityManager.HttpMethods.md#entitytransformchanged]]

### EntityHitZone

**Сигнатура**
```
class EntityHitZone
{
	string name;
	float healthScaled;
}
```

**Назначение**
Одна HitZone экземпляра в разреженном оверлее относительно целого префаба. Соответствует `EntityHitZoneDto`. Ключ на бекенде: `(entityUid, name)`. Хранятся только зоны с `healthScaled < 1`; пустой список оверлея — сущность как из префаба.

**Свойства**
- `name: string` - стабильное имя зоны в префабе (`HitZone.GetName()`).
- `healthScaled: float` - здоровье зоны в диапазоне `0..1` (`HitZone.GetHealthScaled()`).

Группа HitZone, `EDamageState`, абсолютный maxHealth и индекс колеса не хранятся: они выводятся из префаба и `healthScaled`.

**Связано**
[[#EntityItem]], [[EntityManager.HttpMethods.md#applyoperations]]

**Используется в**
[[#EntityItem]], [[EntityManager.md#enqueueentityhitzonechanged]]

### EntityItem

**Сигнатура**
```
class EntityItem
{
	string uid;
	string prefabName;
	bool isContainer;
	Position position;
	Angle angle;
	string parentContainerUid;
	string inventorySlotUid;
	string ownerCharacterUid;
	string ownerUid;
	StorageTypeEnum storageType;
	EntityHitZone[] hitZones;
}
```

**Назначение**
Сущность игрового мира в реестре (соответствует `EntityItemDto`). Элемент плоского списка из [[EntityManager.HttpMethods.md#getInventoryByCharacterUid]] и [[EntityManager.HttpMethods.md#findEntitiesByUidList]]. Для техники `uid` совпадает с [[VehicleService.Entities.md#Vehicle|Vehicle.entityUid]]; последний водитель и выборка техники — [[VehicleService.md|VehicleService]], не это поле. Оверлей HitZone — поле реестра сущности, не VehicleService.

**Свойства**
- `uid: string` - идентификатор сущности.
- `prefabName: string` - имя префаба сущности.
- `isContainer: bool` - признак контейнера (в него могут быть вложены другие сущности).
- `position: Position|null` - позиция в мире; `null`, если сущность не лежит в мире (в контейнере / слоте).
- `angle: Angle|null` - угол в мире; `null`, если сущность не лежит в мире (в контейнере / слоте). Заполнен тогда же, когда `position`.
- `parentContainerUid: string|null` - идентификатор родительского контейнера; `null`, если нет родителя-контейнера.
- `inventorySlotUid: string|null` - идентификатор слота инвентаря / экипировки; `null`, если слот не задан.
- `ownerCharacterUid: string|null` - идентификатор персонажа, у которого сущность в инвентаре / экипировке; `null`, если не привязана к персонажу. Не путать с `ownerUid`.
- `ownerUid: string|null` - владелец для доступа в [[SafeZoneService.md|safe-zone]] (`TSM_EntityProps.ownerUid`); `null`, если не задан. Слой safe-zone при `null` доступ не запрещает.
- `storageType: StorageTypeEnum` - тип хранилища сущности (не null).
- `hitZones: EntityHitZone[]|null` - разреженный оверлей HitZone. В [[EntityManager.HttpMethods.md#findEntitiesByUidList]] — список повреждённых зон (может быть пустым); в [[EntityManager.HttpMethods.md#getInventoryByCharacterUid]] не отдаётся (`null` / поле отсутствует).

Занятость слота считается по тройке: корневой слот персонажа — `ownerCharacterUid` + `inventorySlotUid` + `storageType`; слот в контейнере — `parentContainerUid` + `inventorySlotUid` + `storageType`. Один и тот же `inventorySlotUid` у персонажа или в контейнере допустим в разных `storageType`. Предметы без слота (`inventorySlotUid` = null) эту уникальность не занимают.

Вложенность восстанавливается по `parentContainerUid` и `inventorySlotUid` (ответ метода — плоский список, не дерево).

**Связано**
[[#Position]], [[#Angle]], [[#StorageTypeEnum]], [[#EntityHitZone]], [[VehicleService.Entities.md#Vehicle]], [[SafeZoneService.md]]

**Используется в**
[[#GetInventoryByCharacterUidResult]], [[#FindEntitiesByUidListResult]]

### EntityParams

**Сигнатура**
```
class EntityParams
{
	string uid;
	string prefabName;
	float radius;
	SpawnPlaceTypeEnum spawnPlaceType;
	string|null description;
	bool isContainer;
	bool isVehicle;
	float weight;
	float volume;
	int slotSizeX;
	int slotSizeY;
	int slotSizeZ;
	float|null maxWeight;
	float|null maxVolume;
	int|null maxSlotSizeX;
	int|null maxSlotSizeY;
	int|null maxSlotSizeZ;
}
```

**Назначение**
Параметры префаба в таблице `entity_params` (соответствует `EntityParamsDto`). Каталог характеристик типа сущности, не экземпляр в мире ([[#EntityItem]]).

Физические поля (`prefabName`, `radius`, `isContainer`, `isVehicle`, `weight`, `volume`, `slotSize*`, `max*`) приходят из команды [[../CommandBus/Commands.md#ParsePrefab|ParsePrefab]] / [[../CommandBus/CommandBus.Entities.md#PrefabDataDto|PrefabDataDto]]. `uid`, `description` и `spawnPlaceType` задаёт бекенд.

Если `isContainer = true`, все `max*` обязательны и неотрицательны.

**Свойства**
- `uid: string` — идентификатор записи; задаёт бекенд.
- `prefabName: string` — имя префаба; уникальный ключ каталога.
- `radius: float` — радиус, который занимает сущность; из `PrefabDataDto`.
- `spawnPlaceType: SpawnPlaceTypeEnum` — тип места для спавна; задаёт бекенд. Значения: `any`, `any_vehicle`, `land_vehicle`, `helicopter_vehicle`.
- `description: string|null` — текстовое описание каталога; задаёт бекенд.
- `isContainer: bool` — признак контейнера; из `PrefabDataDto`.
- `isVehicle: bool` — признак техники; из `PrefabDataDto`.
- `weight: float` — вес сущности; из `PrefabDataDto`.
- `volume: float` — объём сущности; из `PrefabDataDto`.
- `slotSizeX: int` — размер сущности по оси X в слотах; из `PrefabDataDto`.
- `slotSizeY: int` — размер сущности по оси Y в слотах; из `PrefabDataDto`.
- `slotSizeZ: int` — размер сущности по оси Z в слотах; из `PrefabDataDto`.
- `maxWeight: float|null` — максимальный вес содержимого контейнера; из `PrefabDataDto`.
- `maxVolume: float|null` — максимальный объём содержимого контейнера; из `PrefabDataDto`.
- `maxSlotSizeX: int|null` — максимальный размер слота контейнера по оси X; из `PrefabDataDto`.
- `maxSlotSizeY: int|null` — максимальный размер слота контейнера по оси Y; из `PrefabDataDto`.
- `maxSlotSizeZ: int|null` — максимальный размер слота контейнера по оси Z; из `PrefabDataDto`.

**Связано**
[[../CommandBus/CommandBus.Entities.md#PrefabDataDto]], [[../CommandBus/Commands.md#ParsePrefab]]

**Используется в**
`EntityRegistryService.getEntityParamsListByPrefabName`

### GetInventoryByCharacterUidResult

**Сигнатура**
```
class GetInventoryByCharacterUidResult
{
	OperationStatusEnum status;
	string failReason;
	EntityItem[] entities;
}
```

**Назначение**
Результат запроса инвентаря персонажа ([[EntityManager.HttpMethods.md#getInventoryByCharacterUid]]). API-обёртка над `list<EntityItemDto>` сервиса `EntityRegistryService`.

**Свойства**
- `status: OperationStatusEnum` - итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]). Для этого метода доменных отказов нет: при успешном вызове сервиса всегда `Ok`.
- `failReason: string|null` - при `Ok` всегда `null`. Отдельный enum причин отказа не вводится: сервис `getInventoryByCharacterUid` не бросает доменных исключений.
- `entities: EntityItem[]|null` - при `Ok`: плоский список сущностей инвентаря / носимого набора персонажа (может быть пустым); при `Fail` — `null` (зарезервировано). Поле `hitZones` у элементов не заполняется.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#EntityItem]]

**Используется в**
[[EntityManager.HttpMethods.md#getInventoryByCharacterUid]]

### FindEntitiesByUidListResult

**Сигнатура**
```
class FindEntitiesByUidListResult
{
	OperationStatusEnum status;
	string failReason;
	EntityItem[] entities;
}
```

**Назначение**
Результат выборки сущностей по списку uid ([[EntityManager.HttpMethods.md#findEntitiesByUidList]]). API-обёртка над `list<EntityItemDto>` сервиса `EntityRegistryService`.

**Свойства**
- `status: OperationStatusEnum` - итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]). При успешном вызове сервиса всегда `Ok`. Доменных отказов нет; request-level `Fail` возможен только при невалидных params (`InvalidParams`).
- `failReason: string|null` - при `Ok` всегда `null`; при request Fail — `"InvalidParams"`.
- `entities: EntityItem[]|null` - при `Ok`: плоский список найденных сущностей и их вложенных потомков (отсутствующие uid пропущены; список может быть пустым; корни — в порядке запрошенных uid, затем потомки); при `Fail` — `null`. У каждого элемента заполнен разреженный `hitZones` (пустой массив — префаб как есть).

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#EntityItem]]

**Используется в**
[[EntityManager.HttpMethods.md#findEntitiesByUidList]]
