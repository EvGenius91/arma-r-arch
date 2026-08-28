# Архитектура бекенда (Laravel)

Документация по структуре и соглашениям Laravel-бекенда игрового сервера.

Связано с доменной архитектурой менеджеров: [[../Architecture/Architecture.md]], HTTP API: [[../api/http api.md]], игровой клиент: [[../ArchGame/ArchGame.md]].

## Принципы

- [[ServiceAndApiLayers.md]] — разделение слоёв Service и API (домен vs транспорт JSON-RPC)

## Сервисы и контракты

Каждый менеджер из [[../Architecture/Architecture.md]] реализуется как сервис с контрактом:

| Менеджер (документация) | Сервис (Laravel) | Контракт |
|-------------------------|------------------|----------|
| TradeManager | TradeService | `Contracts/TradeService/` |
| BankingManager | BankingService | `Contracts/BankingService/` |
| GameManager | GameService | `Contracts/GameService/` — RPC `game@gameStarted` ([[../Architecture/GameManager.md]], [[../Architecture/GameManager.Entities.md]]) |
| EntityManager | EntityRegistryService | `Contracts/EntityRegistryService/` — RPC `entity@getInventoryByCharacterUid`, `entity@findEntitiesByUidList`, `entity@applyOperations` ([[../Architecture/EntityManager.HttpMethods.md]], [[../Architecture/EntityManager.Entities.md]]) |
| CommandBus | CommandService | `Contracts/CommandService/` — RPC `command@getPendingCommands`, `command@reportCommands` ([[../CommandBus/CommandBus.HttpMethods.md]], [[../CommandBus/CommandBus.Entities.md]]) |

Структура контракта сервиса:

```
Contracts/{ServiceName}/
├── {ServiceName}Interface.php
├── Dto/
├── Enums/
└── Exceptions/
```

Общие enum (например `OperationStatus`) — в `Contracts/Common/Enums/`.

Реализация — в `app/Services/{ServiceName}/`. Eloquent-модели — в `app/Models/`, не в Contracts.
