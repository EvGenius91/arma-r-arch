# Команды CommandBus

Каталог команд, которые бекенд ставит в очередь для игрового сервера. CommandBus забирает пачку, маршрутизирует в целевой сервис; сервис после выполнения вызывает [[CommandBus.md#reportcommands|CommandBus::reportCommands]] со статусом `Completed` или `Fail` (+ `FailReason`).

---

### SpawnEntity

**Маршрут:** CommandBus → [[../Architecture/EntityManager.md|EntityManager]] (`enqueueCommand`).

**Описание:**

Заспавнить сущность (предмет, технику и т.п.) в игровом мире по данным с бекенда.

Когда команды приходят с бекенда, CommandBus передаёт их в EntityManager. EntityManager **одним запросом** получает данные по сущностям ([[../Architecture/EntityManager.HttpMethods.md#findEntitiesByUidList|entity@findEntitiesByUidList]]) и спавнит их в мире. После этого отправляет отчёт по **каждой** команде в `CommandBus::reportCommands`.

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена на бекенде
- Нельзя заспавнить сущность в мире (нет места / недопустимое расположение)

---

### removeEntity

**Маршрут:** CommandBus → [[../Architecture/EntityManager.md|EntityManager]] (`enqueueCommand`).

**Описание:**

Удалить сущность из мира, отметить в реестре как удалённую. Выполняется даже при локальном lock uid ([[../Architecture/EntityLockRegistry.md|EntityLockRegistry]]); после удаления — Unlock и очистка очереди ops по uid.

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена

---

### removeVehicle

**Маршрут:** CommandBus → [[../Architecture/EntityManager.md|EntityManager]] (`enqueueCommand`).

**Описание:**

Удалить технику из мира, отметить в реестре как удалённую.

**Возможные причины невыполнения (`FailReason`):**

- Техника с заданным uid не найдена

---

## См. также

- [[CommandBus.md]] — опрос бекенда, маршрутизация, `reportCommands`
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — `enqueueCommand`, спавн и удаление сущностей
