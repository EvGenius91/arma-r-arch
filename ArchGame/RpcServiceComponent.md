# RPC-компонент фасада сервисов

Правило создания компонентов, которые существуют на клиенте и игровом сервере и проксируют вызовы сервисов через RPC.

Эталон: `TSM_ShopMenuComponent`.

## Назначение

Компонент **не содержит бизнес-логику**. Он отвечает только за:

- публичный API для UI и других компонентов;
- RPC-мост клиент ↔ сервер;
- делегирование в сервисы из `TSM_ServiceContainer`.

Бизнес-логика — в `Services/*`, контракты — в `Contracts/Services/*`.

## Структура файла

Один файл `{Name}Component.c`. Компонент наследуется напрямую от `ScriptComponent`. Код разбит на секции с разделителем:

```enforce
// ---------------------------------------------------------------------------
// {Название секции}
// ---------------------------------------------------------------------------
```

Порядок секций:

1. `{Name}ComponentClass : ScriptComponentClass {}`
2. `{Name}Component : ScriptComponent` — основной класс:
   - **Поля** — контекст, сервисы, `m_Last{Operation}Callback`, `m_{Operation}ServerCallback`
   - **Инициализация** — `OnPostInit`
   - **Контекст** — setters (`SetCharacterUid`, `SetShopUid`)
   - **{Operation}** — на каждую async-операцию: публичный метод, `Ask{Operation}`, `Answer{Operation}`, `SendAnswer{Operation}`
3. **Server adapter-классы** — `{Name}Component{Operation}Callback` после закрытия основного класса

Пример: `TSM_ShopMenuComponent.c` — секции `GetCash`, `GetInventoryForSell`, затем adapter-классы.

## Поля компонента

| Поле | Назначение |
|------|------------|
| `m_CharacterUid`, `m_ShopUid` | Контекст операций; uid задаётся через setter до вызова методов |
| `m_{Service}Interface` | Сервисы из `TSM_ServiceContainer.GetInstance()` |
| `m_Last{Operation}Callback` | Pending callback клиента (один запрос за раз) |
| `m_{Operation}ServerCallback` | Переиспользуемый server-adapter; создаётся один раз в `OnPostInit` |

Именование: `m_` + camelCase. Все поля — с русским `/**! */` описанием.

## Инициализация

```enforce
override event protected void OnPostInit(IEntity owner)
{
    TSM_ServiceContainer container = TSM_ServiceContainer.GetInstance();
    m_TradeService = container.GetTradeService();
    m_BankingService = container.GetBankingService();
    m_GetCashServerCallback = new TSM_ShopMenuComponentGetCashCallback(this);
}
```

Сервисы резолвятся из контейнера. Server-callback создаётся один раз, не на каждый запрос.

## Паттерн одной async-операции

Для операции `{Operation}` (например `GetCash`):

### 1. Публичный метод (клиент)

```enforce
void GetCash(TSM_BankingManagerGetCashCallback callback)
```

- `callback == null` → `Print(..., LogLevel.ERROR)`, выход
- `m_CharacterUid.IsEmpty()` → `Print(..., LogLevel.WARNING)`, выход, **callback не вызывать**
- `m_LastGetCashCallback` уже занят → выход (защита от параллельных запросов)
- иначе: сохранить callback, `Rpc(AskGetCash)`

### 2. RPC Ask (сервер)

```enforce
[RplRpc(RplChannel.Reliable, RplRcver.Server)]
void AskGetCash()
```

- `m_CharacterUid` пустой → `Print(..., LogLevel.WARNING)`, выход
- иначе: `m_BankingService.GetCash(m_CharacterUid, m_GetCashServerCallback)`

### 3. Server adapter (отдельный класс)

```enforce
class TSM_ShopMenuComponentGetCashCallback : TSM_BankingManagerGetCashCallback
{
    override void OnSuccess(TSM_BankingManagerGetCashResultDto result)
    {
        m_ShopMenuComponent.SendAnswerGetCash(
            result.GetStatus(),
            result.GetFailReason(),
            result.GetAmount()
        );
    }
}
```

Adapter вызывает `SendAnswer{Operation}` — публичную обёртку `Rpc(Answer{Operation}, ...)`. Прямой вызов `Rpc(Answer{Operation}, ...)` из внешнего класса недоступен: методы с `[RplRpc]` effectively protected в Enforce.

### 4. RPC Answer (клиент-владелец)

Методы `Answer{Operation}` и `SendAnswer{Operation}` — в секции `{Operation}` основного класса.

```enforce
[RplRpc(RplChannel.Reliable, RplRcver.Owner)]
void AnswerGetCash(
    TSM_OperationStatusEnum status,
    TSM_BankingManagerGetCashFailReasonEnum failReason,
    int amount
)
```

- `m_LastGetCashCallback` пуст → выход
- собрать DTO из примитивов
- `m_LastGetCashCallback.OnSuccess(dto)`
- `m_LastGetCashCallback = null`

```enforce
void SendAnswerGetCash(
    TSM_OperationStatusEnum status,
    TSM_BankingManagerGetCashFailReasonEnum failReason,
    int amount
)
{
    Rpc(AnswerGetCash, status, failReason, amount);
}
```

## Именование RPC

| Роль | Имя | Атрибут |
|------|-----|---------|
| Запрос клиент → сервер | `Ask{Operation}` | `RplRcver.Server` |
| Ответ сервер → клиент | `Answer{Operation}` | `RplRcver.Owner` |

Канал: `RplChannel.Reliable`.

Не использовать префиксы `Rpc_`, `Client_`, `Server_` — только `Ask*` / `Answer*`.

## RPC-параметры

Через RPC передавать **только сериализуемые типы** (enum, int, string, array и т.д.).

DTO целиком в `[RplRpc]` не передавать — разбирать на поля и собирать на принимающей стороне.

## Callback и DTO

- Callback — из `Contracts/Services/{Service}/Callbacks/`
- DTO результата — из `Contracts/Services/{Service}/Dto/`, иммутабельный, суффикс `Dto`
- Результат всегда через `callback.OnSuccess(resultDto)`, не через `return`
- FailReason — enum из `Contracts/Services/{Service}/FailReasons/`

## Разделение ответственности

| Слой | Ответственность |
|------|-----------------|
| UI | Вызывает компонент, реализует client-callback |
| Компонент | RPC, валидация контекста, pending callback |
| Сервис | Бизнес-логика, работа с бекендом |
| Contracts | Интерфейсы, DTO, callback, enum |

Компонент **не** ходит в бекенд напрямую. Сервис **не** делает RPC.

## Логирование

Формат: `[TSM][{ComponentShortName}] {Method}: {message}`

| Ситуация | Уровень |
|----------|---------|
| `callback == null` | `LogLevel.ERROR` |
| `m_CharacterUid` не задан | `LogLevel.WARNING` |

При WARNING callback не вызывать.

## Чеклист новой операции

- [ ] Метод в `Contracts/Services/*/...Interface`
- [ ] Callback + ResultDto + FailReason enum в Contracts
- [ ] Реализация в `Services/*`
- [ ] Регистрация сервиса в `TSM_ServiceContainer` (если новый)
- [ ] `void {Operation}(callback)` — публичный метод компонента
- [ ] `Ask{Operation}` + `Answer{Operation}` + `SendAnswer{Operation}` — RPC в секции `{Operation}`
- [ ] `m_Last{Operation}Callback` + `m_{Operation}ServerCallback`
- [ ] `{Name}Component{Operation}Callback` — adapter-класс
- [ ] Русские описания у класса, полей и методов

## Поток (шаблон)

```
UI → Component.{Operation}(callback)
  → Rpc(Ask{Operation})
    → Service.{Operation}(uid, serverCallback)
      → SendAnswer{Operation}(...primitives)
        → Rpc(Answer{Operation}, ...primitives)
          → callback.OnSuccess(dto)
```
