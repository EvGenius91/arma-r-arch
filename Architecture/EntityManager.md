# EntityManager

Назад: [[Architecture.md]]

EntityManager хранит состояние всех сущностей в игре: техники, оружия, боеприпасов, одежды, экипировки, предметов инвентаря и других объектов, которые могут существовать в мире или быть привязаны к игроку.

## Зависимости

```text
EntityManager ──depends on──► EntityLockRegistry
```

[[EntityLockRegistry.md|EntityLockRegistry]] — **отдельный сервис** (описан в своём документе). EntityManager не хранит реестр блокировок внутри себя: `Lock` ставится **когда пачка ops по uid уходит на бекенд (in-flight)**, после ответа — `Unlock`. Пока ops только в очереди — lock нет. Политика очереди / flush / barrier: [[EntityManager.Operations.md]].

Тем же сервисом пользуется TradeManager (`SellPending`), поэтому in-flight дропа и продажа видят одну картину занятости uid.

## Назначение

EntityManager является реестром игровых сущностей и связывает состояние бекенда с состоянием игрового мира.

Он отвечает за:

- регистрацию новых сущностей в БД;
- выдачу и сохранение `uid` сущности;
- хранение текущего положения сущности: в мире, у игрока или внутри контейнера / в слоте;
- **синхронизацию с бекендом** изменений владения и расположения (подбор, дроп, move, экипировка, передача);
- заполнение инвентаря нового персонажа при спавне (`loadInventory`);
- вызов [[EntityLockRegistry.md|EntityLockRegistry]] на время in-flight пачки (блок / разблок uid);
- обновление связей при изменении инвентаря;
- регистрацию удаления сущности;
- приём команд с [[../CommandBus/CommandBus.md|CommandBus]] через `enqueueCommand` (путь бекенд → мир) и обязательный отчёт через `CommandBus::reportCommands`.

Бекенд остается источником истины по состоянию сущностей. Игровой сервер применяет решения бекенда в мире отдельным шагом через CommandBus. Подробнее: [[BackendGameMutation.md]].

### Предметы — сущности мира

Элементы инвентаря и экипировки — **сущности в игровом мире**. К инвентарю персонажа они привязаны только связями:

- владелец (персонаж) — при необходимости;
- контейнер (`containerUid`);
- слот (экипировка / позиция в контейнере);
- либо координаты в мире (если предмет лежит на земле).

Предмет **не перестаёт** быть сущностью мира, когда попадает в рюкзак или слот оружия. Поэтому синхронизация предметов с бекендом — **ответственность EntityManager**, а не [[CharacterStateManager (черновик) 3.md|CharacterStateManager]].

В снимке CharacterState инвентарь — лишь **проекция** связей из этого реестра (для UI / recovery). CSM не принимает и не шлёт item/equip operations.

---

## Спавн лута

Спавн лута начинается на стороне игрового сервера. Когда игрок подходит к зданию или другому лутабельному объекту, внутриигровой `LootSpawnManager` определяет, какие предметы должны появиться и в каких позициях.

`LootSpawnManager` отправляет в EntityManager запрос со списком элементов и координатами `x`, `y`, `z`. EntityManager регистрирует эти элементы на бекенде и назначает каждому `uid`. Бекенд ставит в очередь команды `SpawnEntity`; [[../CommandBus/CommandBus.md|CommandBus]] доставляет их в EntityManager (`enqueueCommand`), который спавнит сущности в мире и отвечает через `CommandBus::reportCommands`.

```mermaid
sequenceDiagram
    participant Player as Игрок
    participant Loot as LootSpawnManager
    participant Entity as EntityManager
    participant BE as Бекенд
    participant CB as CommandBus
    participant World as Игровой_мир

    Player->>Loot: подходит_к_лутабельному_объекту
    Loot->>Entity: запрос_спавна_items_positions
    Entity->>BE: регистрация_сущностей
    BE-->>Entity: uid_для_сущностей
    BE->>CB: SpawnEntity_в_очереди
    CB->>Entity: enqueueCommand_SpawnEntity
    Entity->>World: создать_сущности_в_мире
    Entity->>CB: reportCommands
    CB->>BE: отчет_о_выполнении
```

---

## Синхронизация расположения (игра → бекенд)

Если игрок берёт предмет, выбрасывает его, передаёт другому игроку, перемещает между контейнерами или надевает / снимает экипировку, **игровые системы инвентаря / мира вызывают EntityManager** (не CharacterStateManager и не `applyOperations` персонажа).

EntityManager отправляет изменение на бекенд и обновляет в реестре связи сущности:

- предмет находится у конкретного игрока;
- предмет лежит в конкретном контейнере / слоте;
- предмет находится в мире на определенной позиции;
- предмет больше не принадлежит игроку или контейнеру.

Так EntityManager сохраняет актуальную картину владения и расположения предметов. Остальные менеджеры опираются на этот реестр при торговле, удалении, восстановлении состояния или сборке проекции CharacterState.

```mermaid
sequenceDiagram
    participant World as Игровой_мир
    participant EM as EntityManager
    participant Lock as EntityLockRegistry
    participant BE as Бекенд
    participant EB as LocalEventBus

    World->>EM: enqueuePickup_Drop_Move
    EM->>Lock: IsLocked
    EM->>EM: реестр_и_очередь
    Note over EM: flush_1с_или_barrier
    EM->>Lock: Lock
    EM->>BE: пачка_ops_по_порядку
    BE->>BE: обновить_реестр
    BE-->>EM: Ok
    EM->>Lock: Unlock
    EM->>EB: InventoryChanged_или_аналог
    Note over BE: при_необходимости_CharacterStateChanged
```

**Граница с CSM:**

| Действие | Кто |
|----------|-----|
| Подбор / дроп / move / equip / unequip / transfer / split-merge / destroy сущности | EntityManager |
| HP, жажда, голод, поза/координаты персонажа, cash вне Trade | CharacterStateManager |
| Buy / Sell | TradeManager → бекенд → CommandBus в мир |

Если commit связей у персонажа на бекенде увеличивает `stateVersion` CharacterState, публикуется `CharacterStateChanged`. CSM только мягко сверяет версию; мир предметов уже согласован через EntityManager / CommandBus.

Подробный жизненный цикл операций и очередь: [[EntityManager.Operations.md]].

---

## Блокировки через EntityLockRegistry

Полное описание сервиса: [[EntityLockRegistry.md]].

**Правило:** блокировка uid для ops EntityManager — **только через EntityLockRegistry**, и **только на время in-flight** (не на всё время очереди).

| Момент | Вызов EntityLockRegistry |
|--------|--------------------------|
| `enqueue*` | `IsLocked`? если да (in-flight / SellPending) — отказ; иначе регистрация расположения + очередь **без** Lock |
| flush пачки (~1 с / barrier) | `Lock(…, ownerService: EntityManager)` → отправка |
| Ok / Fail с бекендом | `Unlock` |
| Уже есть `SellPending` от Trade | отказ enqueue (не выбрасывать продаваемый предмет) |
| CommandBus `removeEntity` | удалить сущность **даже при lock** → `Unlock` → очистить очередь ops по uid |
| Hard-reset дюпов | выровнять мир → `Unlock` → очистить очередь |

```text
enqueuePickupEntity / enqueueDropEntity / enqueueMoveEntity
  → EntityLockRegistry.IsLocked (отказ если занято)
  → зафиксировать уже случившееся в мире расположение в реестре + очередь EntityManager
     (для PickUp мир двигает IEntity сам; EM не клонирует и не перекладывает предмет повторно)
  → flush (таймер / barrier) → Lock → пачка на бекенд
  → Ok/Fail → Unlock (при Fail — откат расположения в мире)
```

Очередь, barrier (магазин и др.), порядок пачки: [[EntityManager.Operations.md]].

---

## Контейнеры и вложенность

Сущность может быть обычным предметом или контейнером. Контейнеры могут быть вложены в другие контейнеры, а сама сущность-контейнер также остается обычной сущностью с собственным `uid`.

Пример вложенности:

```text
Игрок
└── Рюкзак
    ├── Обувь
    ├── Пистолет
    └── Банка тушенки
```

В этой модели рюкзак принадлежит игроку, а обувь, пистолет и банка тушенки находятся внутри рюкзака. Если рюкзак передан другому игроку или выброшен в мир, EntityManager обновляет связь для рюкзака и сохраняет вложенную структуру его содержимого.

---

## Каталог операций расположения (игра → бекенд)

Контракт бекенда (`EntityRegistryService`). На игровом сервере им соответствуют методы EntityManager; вызывающий код (инвентарь / мир) **не** ходит в CSM. Чтение: [[EntityManager.HttpMethods.md#getInventoryByCharacterUid|entity@getInventoryByCharacterUid]], [[EntityManager.HttpMethods.md#findEntitiesByUidList|entity@findEntitiesByUidList]]. HTTP-пачка при flush: [[EntityManager.HttpMethods.md|entity@applyOperations]].

| Операция (бек) | Смысл |
|----------------|--------|
| `PickUpEntity` | из мира → контейнер / слот персонажа |
| `DropEntity` | из инвентаря / слота → мир |
| `MoveEntity` | между слотами / контейнерами |
| `EquipItem` / `UnequipItem` | надеть / снять |
| `SwapEquipment` | обмен слотов |
| `TransferEntity` | передача другому персонажу |
| `SplitStack` / `MergeStack` | стеки (если есть в проекте) |
| `DestroyEntity` | уничтожение / снятие с реестра (контракт уточняется отдельно) |

Публичная точка входа на игровом сервере — методы `enqueue*` (см. ниже): регистрация уже произошедшего в мире изменения + очередь; `Lock` — при flush пачки. Вызывающий код (инвентарь / мир) **не** ходит в CSM и не собирает HTTP-пачки сам.

---

## Методы

### enqueueCommand

**Сигнатура:**

```text
enqueueCommand(command): void
```

**Параметры:**

- `command` — команда CommandBus, назначенная EntityManager (`SpawnEntity`, `removeEntity`, `removeVehicle` и т.п.; см. [[../CommandBus/Commands.md]]).

**Когда вызывается:**

[[../CommandBus/CommandBus.md|CommandBus]] после получения пачки с бекенда маршрутизирует команды сущностей в EntityManager.

**Описание:**

1. Принимает команды, назначенные EntityManager, и ставит их в очередь на выполнение.
2. Для `SpawnEntity`: одним запросом получает данные сущностей ([[EntityManager.HttpMethods.md#findEntitiesByUidList|entity@findEntitiesByUidList]]) и спавнит их в мире.
3. После выполнения **обязан** вызвать [[../CommandBus/CommandBus.md#reportcommands|CommandBus::reportCommands]] со статусом по каждой команде: `Completed` или `Fail` (+ `FailReason`).

Это канал **бекенд → мир**. Не путать с `enqueuePickupEntity` / `enqueueDropEntity` / `enqueueMoveEntity` (синк **игра → бекенд**).

---

### loadInventory

**Сигнатура:**

```text
loadInventory(string characterUid): void
```

**Параметры:**

- `characterUid` — персонаж, чей инвентарь нужно загрузить и материализовать в мире.

**Когда вызывается:**

После создания нового персонажа в мире — респавн после подключения, возрождение после гибели.

```text
Спавн нового персонажа (респавн после подключения / возрождение после гибели)
  → EntityManager.loadInventory(characterUid)
```

**Описание:**

1. Сам запрашивает инвентарь с бекенда через [[EntityManager.HttpMethods.md#getInventoryByCharacterUid|entity@getInventoryByCharacterUid]].
2. По плоскому списку `EntityItem` создаёт/привязывает сущности к персонажу в мире: `prefabName`, стабильный `entityUid`, контейнер/слот, `storageType`; вложенность восстанавливает по `parentContainerUid` / `inventorySlotUid`.
3. Обновляет локальный реестр EntityManager.

Новый персонаж пустой: `loadInventory` заполняет его снимком с бекенда. Union / merge двух текущих списков инвентаря **запрещён**.

**Чего не делает:**

- не шлёт `applyOperations` / `enqueue*` (это не синк игра → бекенд);
- не является горячим путём после buy/sell / `CharacterStateChanged`;
- не подменяет CommandBus для выдачи **новых** предметов во время игры (`SpawnEntity`).

Заполнение мира предметами при спавне персонажа — только `loadInventory`. Выдача / спавн во время игры — через CommandBus `SpawnEntity` → `enqueueCommand`.

---

### enqueueDropEntity

**Сигнатура:**

```text
enqueueDropEntity(string entityUid, vector position, string characterUid, StorageTypeEnum storageType): void
```

**Параметры:**

- `entityUid` — идентификатор сущности, которую выбрасывают в мир;
- `position` — координаты дропа в мире;
- `characterUid` — персонаж, инициировавший дроп;
- `storageType` — тип хранилища ([[EntityManager.Entities.md#StorageTypeEnum]]), уходит в `payload` операции.

**Описание:**

1. Проверяет, что сущность существует и действие допустимо; через EntityLockRegistry: при `IsLocked(entityUid)` (in-flight EM или `SellPending` от Trade) — отказ.
2. Локально применяет оптимизм: сущность переносится на землю (не клон), обновляются связи в реестре сущностей EntityManager.
3. Добавляет в исходящую очередь операцию `DropEntity` с текущим `resetGeneration` сущности (без Lock).
4. Отправка — на flush (~1 с или barrier): тогда `Lock(entityUid, reason: Drop, …)` и пачка на бекенд. См. [[EntityManager.Operations.md]].

После Ok — `Unlock`. После Fail — откат локального расположения и `Unlock`.

---

### enqueuePickupEntity

**Сигнатура:**

```text
enqueuePickupEntity(string entityUid, string|null targetContainerUid, string characterUid, StorageTypeEnum storageType, string slot?): void
```

**Параметры:**

- `entityUid` — идентификатор подбираемой сущности;
- `targetContainerUid` — контейнер назначения (рюкзак и т.п.); `null`, если подбор в корневой слот экипировки персонажа (рюкзак, штаны, куртка);
- `characterUid` — персонаж, который подбирает;
- `storageType` — тип хранилища назначения ([[EntityManager.Entities.md#StorageTypeEnum]]);
- `slot` — слот назначения; обязателен, когда `targetContainerUid` = `null`.

**Описание:**

1. Проверяет доступность сущности; если `EntityLockRegistry.IsLocked(entityUid)` (in-flight / SellPending) — отказ. Если чужой `Drop` ещё только в очереди — подбор допустим и встаёт в очередь того же uid после `Drop`.
2. **Не переносит** сущность в мире: перенос уже выполнил инвентарь / игровой хук (вызывающий код зовёт `enqueuePickupEntity` *после* факта подбора). EntityManager обновляет связи в локальном реестре под новое расположение и сохраняет снимок прежнего для возможного отката.
3. Ставит в очередь операцию `PickUpEntity` с `resetGeneration` (без Lock).
4. На flush: `Lock(entityUid, reason: PickUp, …)`; если задан `targetContainerUid` — узкий `ContainerDispose` на контейнере назначения по политике; при подборе в корневой слот (`targetContainerUid` = `null`) родительского lock нет; пачка на бекенд с сохранением порядка.

Ok → `Unlock`. Fail → откат в предыдущее расположение (например обратно на землю / в машину) и `Unlock`.

---

### enqueueMoveEntity

**Сигнатура:**

```text
enqueueMoveEntity(string entityUid, string|null targetContainerUid, string characterUid, StorageTypeEnum storageType, string slot?): void
```

**Параметры:**

- `entityUid` — перемещаемая сущность;
- `targetContainerUid` — контейнер назначения; `null`, если цель — корневой слот экипировки персонажа;
- `characterUid` — персонаж, инициировавший перемещение;
- `storageType` — тип хранилища назначения ([[EntityManager.Entities.md#StorageTypeEnum]]);
- `slot` — слот назначения; обязателен, когда `targetContainerUid` = `null`.

**Описание:**

1. Проверяет EntityLockRegistry (`IsLocked` → отказ) и доменные правила (вместимость, владение контейнером и т.п.).
2. Локально обновляет связи контейнер/слот.
3. Ставит в очередь `MoveEntity` с `resetGeneration` (без Lock).
4. На flush: `Lock(entityUid, reason: Move, …)`; если цель — рюкзак / носимый контейнер — дополнительно `ContainerDispose` по политике вложенности (рюкзак — да; машину для езды — нет); пачка на бекенд.

Ok → снять все lock’и этой операции. Fail → откат связей и Unlock.

---

## Дюпы и hard-reset

При нескольких экземплярах одного `entityUid` в мире бекенд инициирует анализатор: hard-reset к истине бекенда и отсечение устаревших операций через `resetGeneration` на сущность.

Подробно: [[EntityManager.DupeAnalyzer.md]].

---

## Анти-паттерны

1. Синхронизировать подбор / дроп / экипировку через CharacterStateManager / `applyOperations` персонажа.
2. Считать инвентарь отдельным списком «вне» реестра сущностей (без `entityUid` и связей контейнер/слот).
3. Менять владение в мире после Trade в обход CommandBus и EntityManager.
4. Делать union / merge двух текущих списков инвентаря при рассинхроне (см. [[CharacterStateManager (черновик) 3.md]]).
5. Дублировать lock внутри EntityManager вместо [[EntityLockRegistry.md|EntityLockRegistry]].
6. Игнорировать `SellPending` от Trade и выбрасывать продаваемый предмет.
7. Глобально стопорить все операции мира из‑за одной сущности.
8. Блокировать вождение машины из‑за in-flight предмета в её инвентаре.
9. Отказать CommandBus `removeEntity` из‑за локального lock.

---

## Связанные документы

- [[Architecture.md]] — оглавление архитектуры менеджеров
- [[BackendGameMutation.md]] — принцип применения изменений мира через CommandBus
- [[EntityLockRegistry.md]] — общий реестр блокировок (EntityManager + Trade)
- [[EntityManager.Entities.md]] — сущности реестра и результаты `getInventoryByCharacterUid` / `findEntitiesByUidList`
- [[EntityManager.Operations.md]] — логика операций и очередь
- [[EntityManager.HttpMethods.md]] — JSON-RPC `entity@getInventoryByCharacterUid`, `entity@findEntitiesByUidList`, `entity@applyOperations`
- [[EntityManager.DupeAnalyzer.md]] — анализатор дюпов, hard-reset, игнор операций
- [[TradeManager.md]] — sell и Lock на время продажи
- [[CharacterStateManager (черновик) 3.md]] — состояние персонажа; инвентарь только как проекция
- [[../api/http api.md]] — транспорт HTTP JSON-RPC
- [[../CommandBus/CommandBus.md]] — доставка команд с бекенда (`run`, `reportCommands`)
- [[../CommandBus/Commands.md]] — каталог команд (`SpawnEntity`, `removeEntity`, …)
