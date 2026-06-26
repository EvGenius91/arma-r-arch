Менеджер банковских операций и денежных средств персонажа: баланс наличных и банковских счетов, банкомат, переводы между игроками (по мере описания в проекте). [[TradeManager.md|TradeManager]] при покупке/продаже опирается на BankingManager для проверки и движения средств.

## Структура документации

- [[BankingManager.Entities.md]]

## Методы

### getCash

**Сигнатура**
getCash(string characterUid): [[BankingManager.Entities.md#GetCashResult]]

**Аргументы**
- `characterUid: string` - персонаж, у которого запрашиваются наличные «в кармане».

**Описание**
Возвращает сумму наличных у персонажа.

**Результат**
- При успехе возвращает `status = Ok` и неотрицательное `amount`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CharacterNotFound`

### getBalanceBankAccount

**Сигнатура**
getBalanceBankAccount(string characterUid, string bankAccountUid): [[BankingManager.Entities.md#GetBalanceResult]]

**Аргументы**
- `characterUid: string` - персонаж, для которого проверяется владение счётом и запрашивается остаток (по правилам проекта).
- `bankAccountUid: string` - идентификатор банковского счёта.

**Описание**
Возвращает остаток на указанном банковском счёте. Наличные у персонажа — через [[#getCash]].

**Результат**
- При успехе возвращает `status = Ok` и неотрицательное `amount`.
- При отказе возвращает `status = Fail` и `failReason`.

**Ошибки / причины отказа**
- `CharacterNotFound`
- `BankAccountNotFound`
- `BankAccountNotOwnedByCharacter`
