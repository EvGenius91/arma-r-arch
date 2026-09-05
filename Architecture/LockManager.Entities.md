## Сущности LockManager

Назад: [[LockManager.md]]

### LockTypeEnum

**Сигнатура**
```
enum LockTypeEnum
{
	case Character;
	case PinCode;
	case KeyEntity;
}
```

**Назначение**
Тип замка: чем проверяется право закрыть или открыть.

**Значения**
- `Character` (`character`) — только персонажи из списка ключей замка.
- `PinCode` (`pinCode`) — по пин-коду.
- `KeyEntity` (`keyEntity`) — в инвентаре актёра должен быть ключ-сущность из списка замка.

**Используется в**
[[LockManager.md]]

### LockResult

**Сигнатура**
```
class LockResult
{
	OperationStatusEnum status;
	LockFailReasonEnum failReason;
}
```

**Назначение**
Результат закрытия замка ([[LockManager.md#lock]]).

**Свойства**
- `status: OperationStatusEnum` — итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]).
- `failReason: LockFailReasonEnum` — причина при `status = Fail`; при `Ok` — не используется.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#LockFailReasonEnum]]

**Используется в**
[[LockManager.md#lock]]

### UnlockResult

**Сигнатура**
```
class UnlockResult
{
	OperationStatusEnum status;
	LockFailReasonEnum failReason;
}
```

**Назначение**
Результат открытия замка ([[LockManager.md#unlock]]).

**Свойства**
- `status: OperationStatusEnum` — итог операции ([[TradeManager.Entities.md#OperationStatusEnum]]).
- `failReason: LockFailReasonEnum` — причина при `status = Fail`; при `Ok` — не используется.

**Связано**
[[TradeManager.Entities.md#OperationStatusEnum]], [[#LockFailReasonEnum]]

**Используется в**
[[LockManager.md#unlock]]

### LockFailReasonEnum

**Сигнатура**
```
enum LockFailReasonEnum
{
	case LockNotFound;
	case ActorRequired;
	case KeyNotFoundInInventory;
	case ActorNotInCharacterKeys;
	case PinCodeMismatch;
}
```

**Назначение**
Причины отказа при закрытии или открытии замка.

**Значения**
- `LockNotFound` — замок с указанным `lockUid` не найден.
- `ActorRequired` — для типа замка `character` или `keyEntity` обязателен `actorCharacterUid`.
- `KeyNotFoundInInventory` — в инвентаре актёра нет ключа-сущности этого замка.
- `ActorNotInCharacterKeys` — персонаж-актёр не прописан в ключах замка типа `character`.
- `PinCodeMismatch` — пин-код не совпадает с заданным в замке (в том числе если не передан для типа `pinCode`).

**Используется в**
[[#LockResult]], [[#UnlockResult]]
