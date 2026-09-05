# SafeZoneService

Назад: [[Architecture.md]] · [[SafeZoneService.Entities.md]]

В **safe-zone** действуют правила доступа к сущностям по `ownerUidList`. Если у сущности задан непустой `ownerUidList`, доступ к ней имеют только персонажи из этого списка — **независимо от состояния замка**.

Бекенд хранит список зон и является источником истины. Игровой сервис `TSM_SafeZoneService` держит локальную копию координат и принимает зоны только из CommandBus. Игра **не** ходит за зонами по RPC.

Доменный контракт бекенда: `SafeZoneService` (`Contracts/SafeZoneService/`). Мир обновляется командами CommandBus `AddSafeZone` и `SetEntityOwner` ([[../CommandBus/Commands.md#AddSafeZone]], [[../CommandBus/Commands.md#SetEntityOwner]]).

Не путать `ownerUidList` с [[EntityManager.Entities.md#EntityItem|EntityItem.ownerCharacterUid]]: `ownerCharacterUid` — кто держит предмет в инвентаре; `ownerUidList` — владельцы для доступа в safe-zone.

## Структура документации

- [[SafeZoneService.Entities.md]] — `TSM_SafeZone`, `TSM_EntityProps.ownerUidList`

## Зависимости

```text
Экшены / инвентарь  ──depends on──►  TSM_EntitySafeZone
TSM_EntitySafeZone  ──depends on──►  TSM_EntityProps
TSM_EntitySafeZone  ──depends on──►  TSM_SafeZoneService
TSM_SafeZoneService ──filled by───►  CommandBus (AddSafeZone)
TSM_EntityProps     ──updated by──►  CommandBus (SetEntityOwner)
```

## Назначение

`TSM_SafeZoneService` хранит список [[SafeZoneService.Entities.md#TSM_SafeZone|TSM_SafeZone]] и отвечает на вопрос, попадает ли точка мира хотя бы в одну зону.

Компонент `TSM_EntitySafeZone` на сущности решает, есть ли у персонажа доступ к этой сущности **в рамках правил safe-zone**. Этот слой **не смотрит** на `isLocked`:

- не-владелец в зоне — доступа нет даже при открытом замке;
- владелец в зоне — доступ есть даже при закрытом замке.

Замок ([[LockManager.md|LockManager]]) проверяется отдельно и только если `isAccess` уже `true`.

Вне зоны, при `ownerUidList == null` или без компонента `TSM_EntitySafeZone` этот слой доступ не режет.

```text
Экшен / инвентарь
  → нет TSM_EntitySafeZone на сущности?  → доступ разрешён (этот слой)
  → TSM_EntitySafeZone.isAccess(characterUid)
       → TSM_EntityProps.ownerUidList == null? → true
       → не в safe-zone?                       → true
       → characterUid в ownerUidList?          → true
       → иначе                                 → false
  → isAccess == false → экшн не показывать, инвентарь не открывать
  → isAccess == true  → дальше обычная проверка замка
```

## Кто вызывает

| Вызывающий | Когда |
|------------|--------|
| Экшены сущности | перед показом: нет `TSM_EntitySafeZone` → показать; `isAccess == false` → не показывать |
| Инвентарь | перед открытием контейнера / сущности: та же проверка через `TSM_EntitySafeZone` |
| `TSM_EntitySafeZone` | `isInSafeZone` у `TSM_SafeZoneService`; `ownerUidList` у `TSM_EntityProps` |
| CommandBus | `AddSafeZone` → `TSM_SafeZoneService`; `SetEntityOwner` → `TSM_EntityProps` на сущности `entityUid` |

---

## TSM_SafeZoneService

Игровой сервис. Хранит список зон в памяти мира. Запись — только команда CommandBus `AddSafeZone`.

### isInSafeZone

**Сигнатура**
isInSafeZone(float x, float y, float z): bool

**Аргументы**
- `x: float` — координата X.
- `y: float` — координата Y.
- `z: float` — координата Z.

**Описание**
Возвращает `true`, если заданные координаты попадают хотя бы в одну safe-zone из списка сервиса.

Попадание: 3D-расстояние от точки до `centerPosition` **строго меньше** `radius` (как `SafeZoneService.isInSafeZone` на бекенде).

**Результат**
- `true` — точка внутри хотя бы одной зоны.
- `false` — ни одна зона не содержит точку.

**Связано**
[[SafeZoneService.Entities.md#TSM_SafeZone]], [[../CommandBus/Commands.md#AddSafeZone]]

### приём AddSafeZone

**Маршрут:** [[../CommandBus/Commands.md#AddSafeZone|AddSafeZone]] → `TSM_SafeZoneService`.

Добавляет [[SafeZoneService.Entities.md#TSM_SafeZone|TSM_SafeZone]] в список сервиса. Аналог `enqueueCommand` у EntityManager: CommandBus передаёт команду, сервис применяет её в мире и отчитывается через `CommandBus::reportCommands`.

Бекенд ставит команду при [[GameManager.md#gameStarted|gameStarted]] для каждой зоны из `SafeZoneService.findAllSafeZones` и при добавлении новой зоны.

---

## TSM_EntityProps

Компонент свойств сущности в мире. Поле `ownerUidList` (`string[]|null`) — владельцы для доступа в safe-zone. Соответствует [[EntityManager.Entities.md#EntityItem|EntityItem.ownerUidList]] на бекенде.

Обновляется только командой CommandBus [[../CommandBus/Commands.md#SetEntityOwner|SetEntityOwner]] (полная замена списка). Бекенд ставит её из `EntityRegistryService.setOwner` (`SetEntityOwnerEvent`) и в одном блоке со `SpawnEntity`, если у спавнящейся сущности `ownerUidList` уже задан и непустой.

`null` — игра не проверяет владельца: слой safe-zone доступ не запрещает.

## TSM_EntitySafeZone

Компонент на сущности, для которой действуют правила safe-zone. Если компонента нет — доступ разрешён (этот слой не применяется).

### isAccess

**Сигнатура**
isAccess(string characterUid): bool

**Аргументы**
- `characterUid: string` — uid персонажа, для которого проверяется доступ.

**Описание**
Через `TSM_EntityProps` получает `ownerUidList`. Через `TSM_SafeZoneService.isInSafeZone` проверяет, находится ли сущность (её позиция в мире) в safe-zone.

Возвращает `false` только если сущность **в** safe-zone **и** `ownerUidList` не `null` и не содержит `characterUid`. Во всех остальных случаях — `true`:

- `ownerUidList == null` — игра не проверяет владельца, доступ разрешён;
- сущность вне safe-zone — доступ разрешён;
- `characterUid` есть в `ownerUidList` — доступ разрешён.

Состояние замка не учитывается.

**Результат**
- `true` — этот слой доступ не запрещает.
- `false` — экшен не показывать, инвентарь не открывать.

**Связано**
[[SafeZoneService.Entities.md#TSM_EntityProps]], [[#isInSafeZone]], [[LockManager.md]]

## См. также

- [[SafeZoneService.Entities.md]] — `TSM_SafeZone`, `TSM_EntityProps.ownerUidList`
- [[../CommandBus/Commands.md#AddSafeZone]] — запись зоны в игровой сервис
- [[../CommandBus/Commands.md#SetEntityOwner]] — обновление `TSM_EntityProps.ownerUidList`
- [[LockManager.md]] — замок проверяется отдельно, после `isAccess`
- [[EntityManager.Entities.md#EntityItem]] — `ownerUidList` в реестре бекенда
- [[GameManager.md#gameStarted]] — постановка `AddSafeZone` при старте сессии
