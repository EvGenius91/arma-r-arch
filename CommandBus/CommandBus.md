# CommandBus

Осуществляет доставку и учёт команд с бекенда на игровом сервере.

Воркер опрашивает бекенд **каждые 0.5 секунды** ([[CommandBus.HttpMethods.md#getPendingCommands|command@getPendingCommands]]) и получает пачку команд, которые нужно выполнить в мире. Каждую команду CommandBus маршрутизирует в целевой сервис в игре (например [[../Architecture/EntityManager.md|EntityManager]]).

Целевой сервис по результатам выполнения обязан вернуть отчёт **по каждой команде**: статус `Completed` или `Fail`; при `Fail` — `FailReason`. После получения отчёта от сервиса CommandBus отправляет отчёт на бекенд ([[CommandBus.HttpMethods.md#reportCommands|command@reportCommands]]).

HTTP-контракт: [[CommandBus.HttpMethods.md]]. Сущности: [[CommandBus.Entities.md]]. Каталог команд: [[Commands.md]].

## Методы

### run

**Сигнатура:**

```text
CommandBus::run(): void
```

Запускает цикл запросов команд с бекенда. Вызывается после инициализации всех систем мира. В цикле каждые 0.5 с вызывает [[CommandBus.HttpMethods.md#getPendingCommands|command@getPendingCommands]] и маршрутизирует полученные команды в целевые сервисы.

### reportCommands

**Сигнатура:**

```text
CommandBus::reportCommands(array reports): void
```

Принимает отчёты о выполнении команд от целевых сервисов. По каждой записи: `Completed` или `Fail` (+ `FailReason` при Fail). После приёма отправляет пачку на бекенд через [[CommandBus.HttpMethods.md#reportCommands|command@reportCommands]].

Элемент `reports` — [[CommandBus.Entities.md#CommandReport]].

## См. также

- [[CommandBus.HttpMethods.md]] — JSON-RPC `command@getPendingCommands`, `command@reportCommands`
- [[CommandBus.Entities.md]] — `Command`, `CommandReport`, Result DTO
- [[Commands.md]] — каталог команд
- [[../Architecture/BackendGameMutation.md]] — принцип: бекенд решает, CommandBus применяет в мире
- [[../Architecture/EntityManager.md]] — целевой сервис для команд сущностей (`enqueueCommand`)
