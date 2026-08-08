# Команды CommandBus

Каталог команд, которые бекенд ставит в очередь для игрового сервера. CommandBus забирает пачку ([[CommandBus.HttpMethods.md#getPendingCommands|command@getPendingCommands]]), маршрутизирует в целевой сервис; сервис после выполнения вызывает [[CommandBus.md#reportcommands|CommandBus::reportCommands]] со статусом `Completed` или `Fail` (+ `FailReason`). Отчёт уходит на бекенд через [[CommandBus.HttpMethods.md#reportCommands|command@reportCommands]].

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

Когда команды приходят с бекенда, CommandBus передаёт их в EntityManager. EntityManager **одним запросом** получает данные по сущностям ([[../Architecture/EntityManager.HttpMethods.md#findEntitiesByUidList|entity@findEntitiesByUidList]]) по `payload.entityUid` и спавнит их в мире. После этого отправляет отчёт по **каждой** команде в `CommandBus::reportCommands`.

**Возможные причины невыполнения (`FailReason`):**

- Сущность с заданным uid не найдена на бекенде
- Нельзя заспавнить сущность в мире (нет места / недопустимое расположение)

---

## См. также

- [[CommandBus.md]] — опрос бекенда, маршрутизация, `reportCommands`
- [[CommandBus.HttpMethods.md]] — JSON-RPC `command@getPendingCommands`, `command@reportCommands`
- [[CommandBus.Entities.md]] — `Command`, `SpawnEntityPayload`, `CommandTypeEnum`, `CommandReport`
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — `enqueueCommand`, спавн сущностей
