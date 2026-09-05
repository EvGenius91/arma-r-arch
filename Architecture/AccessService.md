# AccessService

Назад: [[Architecture.md]] · [[AccessService.Entities.md]]

Игровой сервис контроля доступа к действиям над сущностью: посадка в технику, инвентарь, открытие/закрытие замка. Бекенд — источник истины по грантам. Карта на игровом сервере обновляется **только** командой CommandBus [[../CommandBus/Commands.md#SetAccessByType|SetAccessByType]]. Игра **не** ходит за грантами по RPC.

Чтение: `TSM_AccessService.IsCan`. Клиент запрашивает проверку через RPC-компонент `TSM_AccessComponent`; карта читается на сервере.

## Структура документации

- [[AccessService.Entities.md]] — `AccessPermissionEnum`, грант (`allowAll` / список персонажей)

## Зависимости

```text
TSM_AccessComponent  ──reads──►  TSM_AccessService.IsCan
TSM_AccessService    ──filled by──►  CommandBus (SetAccessByType)
```

## Назначение

`TSM_AccessService` хранит карту: `entityUid` → тип разрешения → грант.

- нет записи по сущности или нет гранта этого типа — `IsCan` = `true` (отсутствие записи ≠ запрет);
- грант с `allowAll` — `true` для любого персонажа;
- грант со списком — `true` только если `actorCharacterUid` в списке (пустой список — никому).

Uid сущности для проверки берётся с `TSM_EntityProps` владельца компонента (сама сущность или корень техники).

## Кто вызывает

| Вызывающий | Когда |
|------------|--------|
| `TSM_AccessComponent` | клиентский `IsCan` → RPC на сервер → `TSM_AccessService.IsCan` |
| CommandBus | `SetAccessByType` → `TSM_AccessService.enqueueCommand` |

---

## TSM_AccessService

Игровой сервис. Карта в памяти мира. Запись — только команда CommandBus `SetAccessByType`.

### IsCan

**Сигнатура**
IsCan(AccessPermissionEnum permission, string actorCharacterUid, IEntity entity): bool

**Аргументы**
- `permission: [[AccessService.Entities.md#AccessPermissionEnum|AccessPermissionEnum]]` — проверяемое право.
- `actorCharacterUid: string` — персонаж, который выполняет действие.
- `entity: IEntity` — сущность, к которой относится действие.

**Описание**
Возвращает `true`, если действие разрешено. Нет карты по uid сущности или нет гранта этого типа — разрешено. Иначе — по гранту (`allowAll` или список).

**Результат**
- `true` — действие разрешено.
- `false` — действие запрещено (или невалидные аргументы).

**Связано**
[[AccessService.Entities.md#AccessGrant]], [[../CommandBus/Commands.md#SetAccessByType]]

### приём SetAccessByType

**Маршрут:** [[../CommandBus/Commands.md#SetAccessByType|SetAccessByType]] → `TSM_AccessService`.

Записывает грант для пары `entityUid` + `permissionType` (полная замена этого типа). Остальные типы у сущности не трогаются. Сущность в мире для записи не требуется.

CommandBus передаёт команду в `TSM_AccessService`; сервис применяет запись и отправляет отчёт в `CommandBus::reportCommands`.

---

## См. также

- [[AccessService.Entities.md]] — enum прав и грант
- [[../CommandBus/Commands.md#SetAccessByType]] — запись гранта в карту
- [[../CommandBus/CommandBus.md]] — доставка команд с бекенда
- [[../ArchGame/RpcServiceComponent.md]] — RPC-фасад `TSM_AccessComponent`
