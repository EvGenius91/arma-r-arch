# Архитектура

Оглавление документации менеджеров игрового сервера.

## Принципы

- [[BackendGameMutation.md]] — как бекенд принимает решение, а CommandBus применяет его в игровом мире (продажа, покупка)

## TradeManager

Менеджер торговли: покупка, продажа, взаимодействие со средствами игрока.

- [[TradeManager.md]] — обзор и быстрая навигация
- [[TradeManager.Entities.md]] — сущности и enum
- [[TradeManager.BuyMethods.md]] — методы покупки
- [[TradeManager.SellMethods.md]] — методы продажи
- [[TradeManager.ShopMethods.md]] — методы магазина

## BankingManager

Менеджер банковских операций и денежных средств персонажа.

- [[BankingManager.md]] — обзор и методы
- [[BankingManager.Entities.md]] — сущности и enum

## EntityManager

Регистрация объектов, создаваемых в мире (техника, предметы инвентаря).

- [[EntityManager.md]] — методы registerEntityBuy, registerRemoveEntity
