## Сущности GameManager

Назад: [[GameManager.md]]

Доменный контракт бекенда: `GameService` (`Contracts/GameService/`). Модель `ServerSession` (`Models/Game/`).

### ServerSession

**Сигнатура**
```
class ServerSession
{
	string uid;
	datetime startedAt;
	datetime stoppedAt;
}
```

**Назначение**
Сессия игрового сервера. Регистрируется при [[GameManager.md#gameStarted]]. Текущая (активная) сессия — единственная запись с `stoppedAt = null`.

**Свойства**
- `uid: string` - идентификатор сессии; приходит с игрового сервера в `serverSessionUid`.
- `startedAt: datetime` - дата и время старта игры; задаётся бекендом в момент первого `gameStarted` для этого uid.
- `stoppedAt: datetime|null` - дата и время остановки игры; `null`, пока сессия активна. Записывается при старте следующей сессии (`gameStarted` с другим uid).

**Используется в**
[[GameManager.md#gameStarted]]

### GameStartedResult

**Сигнатура**
```
class GameStartedResult
{
	OperationStatusEnum status;
	string failReason;
}
```

**Назначение**
Результат уведомления о старте игры ([[GameManager.md#gameStarted]]). API-обёртка над `void` сервиса `GameService`.

**Свойства**
- `status: OperationStatusEnum` - итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]). Для этого метода доменных отказов нет: при успешном вызове сервиса всегда `Ok`.
- `failReason: string|null` - при `Ok` всегда `null`. Отдельный enum причин отказа не вводится: сервис `gameStarted` не бросает доменных исключений. Request-level `Fail` возможен только при невалидных params (`InvalidParams`).

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#ServerSession]]

**Используется в**
[[GameManager.md#gameStarted]]
