## Сущности BankingManager

Назад: [[BankingManager.md]]

### GetCashResult

**Сигнатура**
```
class GetCashResult
{
	OperationStatusEnum status;
	GetCashFailReasonEnum failReason;
	int amount;
}
```

**Назначение**
Результат запроса наличных у персонажа ([[BankingManager.md#getCash]]).

**Свойства**
- `status: OperationStatusEnum` - итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]).
- `failReason: GetCashFailReasonEnum` - причина при `status = Fail`.
- `amount: int` - сумма наличных в минимальных денежных единицах мира / условных единицах (как у [[TradeManager.Entities.md#ShopProduct]]) при `status = Ok`; при `Fail` не используется.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#GetCashFailReasonEnum]]

### GetCashFailReasonEnum

**Сигнатура**
```
enum GetCashFailReasonEnum
{
	case CharacterNotFound;
}
```

**Назначение**
Причины отказа при запросе наличных.

**Значения**
- `CharacterNotFound` - персонаж с указанным `characterUid` не найден или недоступен.

**Используется в**
[[#GetCashResult]]

### GetBalanceResult

**Сигнатура**
```
class GetBalanceResult
{
	OperationStatusEnum status;
	GetBalanceFailReasonEnum failReason;
	int amount;
}
```

**Назначение**
Результат запроса остатка на банковском счёте ([[BankingManager.md#getBalanceBankAccount]]).

**Свойства**
- `status: OperationStatusEnum` - итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]: тот же перечислимый тип статуса, что в TradeManager).
- `failReason: GetBalanceFailReasonEnum` - причина при `status = Fail`.
- `amount: int` - остаток на банковском счёте в минимальных денежных единицах мира / условных единицах (как цены в [[TradeManager.Entities.md#ShopProduct]]) при `status = Ok`; при `Fail` не используется.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#GetBalanceFailReasonEnum]]

**Используется в**
[[BankingManager.md#getBalanceBankAccount]]

### GetBalanceFailReasonEnum

**Сигнатура**
```
enum GetBalanceFailReasonEnum
{
	case CharacterNotFound;
	case BankAccountNotFound;
	case BankAccountNotOwnedByCharacter;
}
```

**Назначение**
Причины отказа при запросе остатка на банковском счёте.

**Значения**
- `CharacterNotFound` - персонаж с указанным `characterUid` не найден или недоступен.
- `BankAccountNotFound` - счёт с указанным `bankAccountUid` не найден или недоступен.
- `BankAccountNotOwnedByCharacter` - счёт не принадлежит указанному персонажу (или недоступен ему по правилам проекта).

**Используется в**
[[#GetBalanceResult]]
