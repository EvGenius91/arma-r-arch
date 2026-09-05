## Сущности AccessService

Назад: [[AccessService.md]]

Игровой контракт: `TSM_AccessServiceInterface`, DTO `TSM_AccessGrantDto`, enum `TSM_AccessPermissionEnum`.

### AccessPermissionEnum

**Сигнатура**
```
enum AccessPermissionEnum
{
	case GetIn;
	case AccessToInventory;
	case AccessToOpenCloseLock;
}
```

**Назначение**
Тип права в карте `TSM_AccessService` и в payload команды [[../CommandBus/Commands.md#SetAccessByType|SetAccessByType]]. JSON-строки совпадают с именами значений.

**Значения**
- `GetIn` — право залезть в технику.
- `AccessToInventory` — право на действия с инвентарём.
- `AccessToOpenCloseLock` — право открыть / закрыть замок.

**Используется в**
[[AccessService.md#IsCan]], [[../CommandBus/CommandBus.Entities.md#SetAccessByTypePayload]]

### AccessGrant

**Сигнатура**
```
class AccessGrant
{
	bool allowAll;
	string[]|null characterUidList;
}
```

**Назначение**
Грант одного типа разрешения для одной сущности. Соответствует `TSM_AccessGrantDto`. Элемент, который CommandBus записывает командой [[../CommandBus/Commands.md#SetAccessByType|SetAccessByType]].

**Свойства**
- `allowAll: bool` — `true`: право выдано всем, `characterUidList` не используется.
- `characterUidList: string[]|null` — uid персонажей с правом; `null` при `allowAll`; пустой массив — никому.

`characterUidList = null` в payload команды задаёт `allowAll = true`. Пустой массив задаёт `allowAll = false` и пустой список.

**Связано**
[[../CommandBus/CommandBus.Entities.md#SetAccessByTypePayload]]

**Используется в**
[[AccessService.md#IsCan]], [[AccessService.md#приём SetAccessByType]]
