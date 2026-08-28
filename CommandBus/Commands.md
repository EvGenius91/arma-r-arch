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

## См. также

- [[CommandBus.md]] — опрос бекенда, маршрутизация, `reportCommands`
- [[CommandBus.HttpMethods.md]] — JSON-RPC `command@getPendingCommands`, `command@reportCommands`
- [[CommandBus.Entities.md]] — `Command`, `SpawnEntityPayload`, `DeleteEntityPayload`, `CommandTypeEnum`, `CommandReport`
- [[../Architecture/BackendGameMutation.md]] — бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — `enqueueCommand`, спавн и удаление сущностей
