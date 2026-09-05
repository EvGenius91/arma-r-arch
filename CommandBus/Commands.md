# Команды CommandBus

Каталог команд, которые бекенд ставит в очередь для игрового сервера. CommandBus забирает пачку блоков ([[CommandBus.HttpMethods.md#getPendingCommandBlocks|command@getPendingCommandBlocks]]) и маршрутизирует команды каждого блока в целевые сервисы **в порядке массива `commands`**. Все отчёты, включая отчёты с результирующим payload, уходят пачкой через [[CommandBus.HttpMethods.md#reportCommands|command@reportCommands]].

---

### SpawnEntity

**Маршрут:** CommandBus → [[../Architecture/EntityManager.md|EntityManager]] (`enqueueCommand`).

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-001",
  "type": "SpawnEntity",
  "payload": {
    "entityUid": "ent-can-001"
  }
}
```

`payload` — [[CommandBus.Entities.md#SpawnEntityPayload]].

**Описание:**

Заспавнить сущность (предмет, технику и т.п.) в игровом мире по данным с бекенда.

Когда команды приходят с бекенда, CommandBus передаёт их в EntityManager. EntityManager **одним запросом** получает данные по сущностям ([[../Architecture/EntityManager.HttpMethods.md#findEntitiesByUidList|entity@findEntitiesByUidList]]) по `payload.entityUid` и спавнит их в мире. После создания экземпляра накладывает разреженный оверлей HitZone из `EntityItem.hitZones`: для каждой записи вызывает `SetHealthScaled` на зоне с совпадающим `HitZone.GetName()`. Имена, которых нет на префабе, пропускаются (журналируются). После этого отправляет отчёт по **каждой** команде в `CommandBus::reportCommands`.

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена на бекенде
- Нельзя заспавнить сущность в мире (нет места / недопустимое расположение)

---

### DeleteEntity

**Маршрут:** CommandBus → [[../Architecture/EntityManager.md|EntityManager]] (`enqueueCommand`).

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-003",
  "type": "DeleteEntity",
  "payload": {
    "entityUid": "ent-can-001"
  }
}
```

`payload` — [[CommandBus.Entities.md#DeleteEntityPayload]].

**Описание:**

Удалить сущность из игрового мира по `payload.entityUid`: из инвентаря, с земли или «у другого». Команда идемпотентна и выполняется **даже при lock**; после удаления EntityManager снимает lock (`Unlock`) и очищает очередь ops по этому uid.

Когда команды приходят с бекенда, CommandBus передаёт их в EntityManager. EntityManager удаляет экземпляр в мире и отправляет отчёт по **каждой** команде в `CommandBus::reportCommands`. Отказ выполнить команду из‑за локального lock запрещён — это не `FailReason`.

**Возможные причины невыполнения (`FailReason`):**

- Нельзя удалить сущность из мира

---

### ParsePrefab

**Маршрут:** CommandBus → сервис анализа префабов игрового сервера.

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-005",
  "type": "ParsePrefab",
  "payload": {
    "prefabList": [
      "Prefabs/Items/example.et"
    ]
  }
}
```

`payload` — [[CommandBus.Entities.md#ParsePrefabPayload]].

**Описание:**

Собрать физические характеристики каждого префаба из `payload.prefabList` для заполнения [[../Architecture/EntityManager.Entities.md#EntityParams|EntityParamsDto]] (`EntityRegistryService`). Игра возвращает поля, которые можно снять с префаба (`radius`, `isContainer`, `isVehicle`, `weight`, `volume`, размеры слота и ёмкость контейнера). `uid`, `description` и `spawnPlaceType` игра не передаёт — их задаёт бекенд. После обработки сервис формирует [[CommandBus.Entities.md#ReportParsePrefabPayload]] и добавляет отчёт в пачку `CommandBus::reportCommands` / [[CommandBus.HttpMethods.md#reportCommands|command@reportCommands]].

Пример успешного результирующего payload:

```json
{
  "prefabData": [
    {
      "prefabName": "Prefabs/Items/example.et",
      "radius": 0.15,
      "isContainer": false,
      "isVehicle": false,
      "weight": 1.5,
      "volume": 2.25,
      "slotSizeX": 1,
      "slotSizeY": 1,
      "slotSizeZ": 2,
      "maxWeight": null,
      "maxVolume": null,
      "maxSlotSizeX": null,
      "maxSlotSizeY": null,
      "maxSlotSizeZ": null
    },
    {
      "prefabName": "Prefabs/Items/Equipment/Backpacks/Backpack_01.et",
      "radius": 0.4,
      "isContainer": true,
      "isVehicle": false,
      "weight": 2.0,
      "volume": 8.0,
      "slotSizeX": 4,
      "slotSizeY": 5,
      "slotSizeZ": 2,
      "maxWeight": 40.0,
      "maxVolume": 30.0,
      "maxSlotSizeX": 6,
      "maxSlotSizeY": 6,
      "maxSlotSizeZ": 4
    }
  ]
}
```

**Возможные причины невыполнения (`FailReason`):**

- Префаб не найден или не может быть загружен
- Нельзя определить характеристики префаба

---

### SetLockEntity

**Маршрут:** CommandBus → игровой сервис замков.

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-006",
  "type": "SetLockEntity",
  "payload": {
    "entityUid": "ent-door-001",
    "lockEntityUid": "lk01",
    "lockType": "character",
    "isLocked": false
  }
}
```

`payload` — [[CommandBus.Entities.md#SetLockEntityPayload]].

**Описание:**

Повесить замок на сущность в игровом мире: привязать `lockEntityUid` к `entityUid`, задать тип замка и текущее `isLocked`.

Бекенд ставит команду в тот же блок, что `SpawnEntity`, при старте сессии ([[../Architecture/GameManager.md#gameStarted]]), если у спавнящейся сущности уже есть замок. Порядок в блоке: сначала `SpawnEntity`, затем при необходимости `SetLockEntity`, затем при необходимости [[#SetEntityOwner]]. Смена открыт/закрыт после RPC — отдельная команда [[#ChangeLockStatus]].

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена в мире
- Нельзя установить замок на сущность

---

### ChangeLockStatus

**Маршрут:** CommandBus → игровой сервис замков.

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-007",
  "type": "ChangeLockStatus",
  "payload": {
    "lockUid": "lk01",
    "isLocked": true
  }
}
```

`payload` — [[CommandBus.Entities.md#ChangeLockStatusPayload]].

**Описание:**

Сменить статус замка в мире: закрыт (`isLocked = true`) или открыт (`isLocked = false`). Замок на сущность уже должен быть установлен ([[#SetLockEntity]]).

Бекенд ставит команду после успешного [[../Architecture/LockManager.md#lock|lock@lock]] / [[../Architecture/LockManager.md#unlock|lock@unlock]]: RPC фиксирует `isLocked` на бекенде, CommandBus применяет его в мире. Если статус на бекенде не изменился (замок уже закрыт / уже открыт), команда не ставится.

**Возможные причины невыполнения (`FailReason`):**

- Замок с заданным uid не найден в мире
- Нельзя сменить статус замка

---

### AddSafeZone

**Маршрут:** CommandBus → [[../Architecture/SafeZoneService.md|TSM_SafeZoneService]].

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-008",
  "type": "AddSafeZones",
  "payload": {
    "safeZoneUid": "sz01",
    "safeZoneName": "Base Alpha",
    "centerPosition": {
      "x": 1200.5,
      "y": 32.0,
      "z": 840.25
    },
    "radius": 50.0
  }
}
```

`payload` — [[CommandBus.Entities.md#AddSafeZonePayload]]. JSON-значение `type` — `AddSafeZones` (`CommandTypeEnum::AddSafeZone`).

**Описание:**

Записать safe-zone в список игрового `TSM_SafeZoneService`: uid, имя, центр и радиус.

Бекенд ставит команду при старте сессии ([[../Architecture/GameManager.md#gameStarted]]) для каждой зоны из `SafeZoneService.findAllSafeZones` и при добавлении новой зоны. CommandBus передаёт команду в `TSM_SafeZoneService`; сервис добавляет [[../Architecture/SafeZoneService.Entities.md#TSM_SafeZone|TSM_SafeZone]] в список и отправляет отчёт в `CommandBus::reportCommands`.

**Возможные причины невыполнения (`FailReason`):**

- Нельзя добавить safe-zone

---

### SetEntityOwner

**Маршрут:** CommandBus → [[../Architecture/SafeZoneService.md#TSM_EntityProps|TSM_EntityProps]] на сущности `payload.entityUid`.

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-009",
  "type": "SetEntityOwner",
  "payload": {
    "entityUid": "ent-crate-001",
    "ownerUidList": ["char-001"]
  }
}
```

`payload` — [[CommandBus.Entities.md#SetEntityOwnerPayload]].

**Описание:**

Задать `TSM_EntityProps.ownerUidList` у сущности в игровом мире. Поле используется правилами доступа в safe-zone ([[../Architecture/SafeZoneService.md|SafeZoneService]]): в зоне доступ имеют только персонажи из этого списка, независимо от замка. Команда передаёт сразу всех владельцев (полная замена списка). `ownerUidList = null` — игра не проверяет владельца.

Бекенд ставит команду из `EntityRegistryService.setOwner` (`SetEntityOwnerEvent`) и в том же блоке, что `SpawnEntity`, если у спавнящейся сущности уже задан непустой `ownerUidList`. Порядок в блоке: сначала `SpawnEntity`, затем при необходимости `SetLockEntity`, затем `SetEntityOwner`.

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена в мире
- Нельзя выставить владельца сущности

---

### SetAccessByType

**Маршрут:** CommandBus → [[../Architecture/AccessService.md|TSM_AccessService]] (`enqueueCommand`).

**Формат в API** (`Command`):

```json
{
  "uid": "cmd-010",
  "type": "SetAccessByType",
  "payload": {
    "entityUid": "ent-vehicle-001",
    "permissionType": "GetIn",
    "characterUidList": ["char-001"]
  }
}
```

`payload` — [[CommandBus.Entities.md#SetAccessByTypePayload]]. JSON-значение `type` — `SetAccessByType`.

**Описание:**

Записать одно разрешение в карту игрового `TSM_AccessService` для пары `entityUid` + `permissionType`. Остальные типы разрешений у той же сущности не меняются.

`characterUidList = null` — право выдано всем (`allowAll`). `characterUidList = []` — никому. Непустой список — только указанным персонажам. `null` и пустой массив **не** эквивалентны.

Команда обновляет только in-memory карту; сущность в мире для выполнения не требуется. Бекенд — источник истины по грантам; игра применяет запись в карте и отчитывается через `CommandBus::reportCommands`.

Проверка `IsCan`: нет записи по сущности или нет гранта этого типа — действие разрешено. Если грант есть — смотрим `allowAll` / список персонажей.

Неизвестный `permissionType` на wire команда в игру не попадает (разбор очереди отбрасывает элемент) — это не `FailReason`.

**Возможные причины невыполнения (`FailReason`):**

- `CannotSetAccess` — пустой `entityUid` или неизвестный тип разрешения после разбора

---

## См. также

- [[CommandBus.md]] — опрос бекенда, маршрутизация и отправка отчётов
- [[CommandBus.HttpMethods.md]] — JSON-RPC `command@getPendingCommandBlocks`, `command@reportCommands`
- [[CommandBus.Entities.md]] — `CommandBlock`, `Command`, payload команд, `CommandTypeEnum`, `BaseReport`
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — `enqueueCommand`, спавн и удаление сущностей
- [[../Architecture/LockManager.md]] — RPC `lock@lock` / `lock@unlock`; мир через `ChangeLockStatus`
- [[../Architecture/SafeZoneService.md]] — `AddSafeZone` в `TSM_SafeZoneService`; `SetEntityOwner` обновляет `TSM_EntityProps.ownerUidList`
- [[../Architecture/AccessService.md]] — `SetAccessByType` обновляет карту разрешений `TSM_AccessService`
