## Сущности SafeZoneService

Назад: [[SafeZoneService.md]]

Доменный контракт бекенда: `SafeZoneService` (`Contracts/SafeZoneService/`), DTO `SafeZoneDto` / `AddSafeZoneDto`, `PositionDto`.

### TSM_SafeZone

**Сигнатура**
```
class TSM_SafeZone
{
	string uid;
	string name;
	Position centerPosition;
	float radius;
}
```

**Назначение**
Безопасная зона в списке игрового `TSM_SafeZoneService` (соответствует `SafeZoneDto` на бекенде). Элемент, который CommandBus записывает командой [[../CommandBus/Commands.md#AddSafeZone|AddSafeZone]].

**Свойства**
- `uid: string` — идентификатор зоны.
- `name: string` — отображаемое имя зоны.
- `centerPosition: [[EntityManager.Entities.md#Position|Position]]` — центр зоны в мире.
- `radius: float` — радиус зоны. Точка внутри зоны, если 3D-расстояние до центра **строго меньше** `radius`.

**Связано**
[[EntityManager.Entities.md#Position]], [[../CommandBus/CommandBus.Entities.md#AddSafeZonePayload]]

**Используется в**
[[SafeZoneService.md#isInSafeZone]], [[SafeZoneService.md#приём AddSafeZone]]

### TSM_EntityProps

**Сигнатура**
```
class TSM_EntityProps
{
	string[]|null ownerUidList;
}
```

**Назначение**
Компонент свойств сущности в игровом мире. Для safe-zone существенно поле `ownerUidList`: владельцы сущности для правил доступа. Соответствует [[EntityManager.Entities.md#EntityItem|EntityItem.ownerUidList]] на бекенде, не `ownerCharacterUid`.

Обновляется командой CommandBus [[../CommandBus/Commands.md#SetEntityOwner|SetEntityOwner]] (полная замена списка).

**Свойства**
- `ownerUidList: string[]|null` — uid персонажей-владельцев для доступа в safe-zone; `null` — игра не проверяет владельца, слой safe-zone доступ не запрещает.

**Связано**
[[EntityManager.Entities.md#EntityItem]], [[../CommandBus/CommandBus.Entities.md#SetEntityOwnerPayload]]

**Используется в**
[[SafeZoneService.md#isAccess]]
