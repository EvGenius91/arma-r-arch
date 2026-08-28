## Сущности EntityManager

Назад: [[EntityManager.md]] · [[EntityManager.HttpMethods.md]]

Доменный контракт бекенда: `EntityRegistryService` (`Contracts/EntityRegistryService/`), DTO `EntityItemDto` / `PositionDto` / `AngleDto`, enum `StorageTypeEnum`.

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
	StorageTypeEnum storageType;
}
```

**Назначение**
Сущность игрового мира в реестре (соответствует `EntityItemDto`). Элемент плоского списка из [[EntityManager.HttpMethods.md#getInventoryByCharacterUid]] и [[EntityManager.HttpMethods.md#findEntitiesByUidList]].

**Свойства**
- `uid: string` - идентификатор сущности.
- `prefabName: string` - имя префаба сущности.
- `isContainer: bool` - признак контейнера (в него могут быть вложены другие сущности).
- `position: Position|null` - позиция в мире; `null`, если сущность не лежит в мире (в контейнере / слоте).
- `angle: Angle|null` - угол в мире; `null`, если сущность не лежит в мире (в контейнере / слоте). Заполнен тогда же, когда `position`.
- `parentContainerUid: string|null` - идентификатор родительского контейнера; `null`, если нет родителя-контейнера.
- `inventorySlotUid: string|null` - идентификатор слота инвентаря / экипировки; `null`, если слот не задан.
- `ownerCharacterUid: string|null` - идентификатор персонажа-владельца; `null`, если владелец не задан.
- `storageType: StorageTypeEnum` - тип хранилища сущности (не null).

Занятость слота считается по тройке: корневой слот персонажа — `ownerCharacterUid` + `inventorySlotUid` + `storageType`; слот в контейнере — `parentContainerUid` + `inventorySlotUid` + `storageType`. Один и тот же `inventorySlotUid` у персонажа или в контейнере допустим в разных `storageType`. Предметы без слота (`inventorySlotUid` = null) эту уникальность не занимают.

Вложенность восстанавливается по `parentContainerUid` и `inventorySlotUid` (ответ метода — плоский список, не дерево).

**Связано**
[[#Position]], [[#Angle]], [[#StorageTypeEnum]]

**Используется в**
[[#GetInventoryByCharacterUidResult]], [[#FindEntitiesByUidListResult]]

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
- `entities: EntityItem[]|null` - при `Ok`: плоский список сущностей инвентаря / носимого набора персонажа (может быть пустым); при `Fail` — `null` (зарезервировано).

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
- `entities: EntityItem[]|null` - при `Ok`: плоский список найденных сущностей и их вложенных потомков (отсутствующие uid пропущены; список может быть пустым; корни — в порядке запрошенных uid, затем потомки); при `Fail` — `null`.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#EntityItem]]

**Используется в**
[[EntityManager.HttpMethods.md#findEntitiesByUidList]]
