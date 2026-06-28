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
| EntityManager | EntityService | `Contracts/EntityService/` |

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
