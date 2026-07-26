# EntityManager

Назад: [[Architecture.md]]

EntityManager хранит состояние всех сущностей в игре: техники, оружия, боеприпасов, одежды, экипировки, предметов инвентаря и других объектов, которые могут существовать в мире или быть привязаны к игроку.

## Зависимости

```text
EntityManager ──depends on──► EntityLockRegistry
```

[[EntityLockRegistry.md|EntityLockRegistry]] — **отдельный сервис** (описан в своём документе). EntityManager не хранит реестр блокировок внутри себя: при `enqueuePickupEntity`, `enqueueDropEntity`, `enqueueMoveEntity` сущность **блокируется через EntityLockRegistry** (`Lock`), после ответа бекенда — `Unlock`.

Тем же сервисом пользуется TradeManager (`SellPending`), поэтому дроп и продажа видят одну картину занятости uid.

## Назначение

EntityManager является реестром игровых сущностей и связывает состояние бекенда с состоянием игрового мира.

Он отвечает за:

- регистрацию новых сущностей в БД;
- выдачу и сохранение `uid` сущности;
- хранение текущего положения сущности: в мире, у игрока или внутри контейнера / в слоте;
- **синхронизацию с бекендом** изменений владения и расположения (подбор, дроп, move, экипировка, передача);
- вызов [[EntityLockRegistry.md|EntityLockRegistry]] при enqueue-операциях (блок / разблок uid);
- обновление связей при изменении инвентаря;
- регистрацию удаления сущности;
- постановку команд в CommandBus, когда изменение должно быть применено в игровом мире (путь бекенд → мир).

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

`LootSpawnManager` отправляет в EntityManager запрос со списком элементов и координатами `x`, `y`, `z`. EntityManager регистрирует эти элементы в БД, назначает каждому элементу `uid` и отправляет в игровую командную шину команды на спавн.

После этого CommandBus доставляет команды игровому серверу. Игровой сервер создает предметы в указанных позициях и присваивает им полученные `uid`.

```mermaid
sequenceDiagram
    participant Player as Игрок
    participant Loot as LootSpawnManager
    participant Entity as EntityManager
    participant DB as БД
    participant CB as CommandBus
    participant World as Игровой_мир

    Player->>Loot: подходит_к_лутабельному_объекту
    Loot->>Entity: запрос_спавна_items_positions
    Entity->>DB: регистрация_сущностей
    DB-->>Entity: uid_для_сущностей
    Entity->>CB: команды_спавна_uid_prefab_position
    CB->>World: создать_сущности_в_мире
    World-->>CB: отчет_о_выполнении
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
    EM->>Lock: IsLocked_Lock
    EM->>EM: оптимизм_и_очередь
    EM->>BE: commit_связей_сущности
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

**Правило:** любой `enqueuePickupEntity` / `enqueueDropEntity` / `enqueueMoveEntity` блокирует затронутый `entityUid` **только через EntityLockRegistry**, не через локальное поле EntityManager.

| Момент | Вызов EntityLockRegistry |
|--------|--------------------------|
| `enqueueDropEntity` / `enqueuePickupEntity` / `enqueueMoveEntity` | `IsLocked` → `Lock(…, ownerService: EntityManager)` |
| Ok / Fail операции с бекендом | `Unlock` |
| Уже есть `SellPending` от Trade | отказ enqueue (не выбрасывать продаваемый предмет) |
| CommandBus `removeEntity` | удалить сущность **даже при lock** → `Unlock` → очистить очередь ops по uid |
| Hard-reset дюпов | выровнять мир → `Unlock` → очистить очередь |

```text
enqueuePickupEntity / enqueueDropEntity / enqueueMoveEntity
  → EntityLockRegistry.IsLocked / Lock
  → локальный оптимизм + очередь EntityManager
  → бекенд → Ok/Fail → EntityLockRegistry.Unlock
```

Очередь и жизненный цикл ops: [[EntityManager.Operations.md]].

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

Контракт бекенда (имена ориентировочные). На игровом сервере им соответствуют методы EntityManager; вызывающий код (инвентарь / мир) **не** ходит в CSM.

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

Публичная точка входа на игровом сервере — методы `enqueue*` (см. ниже): ставят операцию в очередь и через [[EntityLockRegistry.md|EntityLockRegistry]] блокируют uid. Вызывающий код (инвентарь / мир) **не** ходит в CSM и не собирает HTTP-пачки сам.

---

## Методы

### enqueueDropEntity

**Сигнатура:**

```text
enqueueDropEntity(string entityUid, vector position, string characterUid): void
```

**Параметры:**

- `entityUid` — идентификатор сущности, которую выбрасывают в мир;
- `position` — координаты дропа в мире;
- `characterUid` — персонаж, инициировавший дроп.

**Описание:**

1. Проверяет, что сущность существует и действие допустимо; через EntityLockRegistry: при конфликтующем `IsLocked(entityUid)` (в т.ч. `SellPending` от Trade) — отказ или постановка в очередь после текущей операции (политика EntityManager).
2. `EntityLockRegistry.Lock(entityUid, reason: Drop, scope: InventoryOps, ownerService: EntityManager)`.
3. Локально применяет оптимизм: сущность переносится на землю (не клон), обновляются связи в реестре сущностей EntityManager.
4. Добавляет в исходящую очередь операцию `DropEntity` с текущим `resetGeneration` сущности.
5. Если по этому uid нет другого in-flight — отправляет операцию на бекенд.

После Ok — `EntityLockRegistry.Unlock(entityUid)`. После Fail — откат локального расположения и `Unlock`.

---

### enqueuePickupEntity

**Сигнатура:**

```text
enqueuePickupEntity(string entityUid, string targetContainerUid, string characterUid, string slot?): void
```

**Параметры:**

- `entityUid` — идентификатор подбираемой сущности;
- `targetContainerUid` — контейнер / слот назначения (инвентарь персонажа, рюкзак и т.п.);
- `characterUid` — персонаж, который подбирает;
- `slot` — слот назначения, если нужен контракту.

**Описание:**

1. Проверяет доступность сущности; если `EntityLockRegistry.IsLocked(entityUid)` — отказ (второй игрок не поднимает банку, пока она занята после чужого дропа/move/sell).
2. `Lock(entityUid, reason: PickUp, scope: InventoryOps, ownerService: EntityManager)`; при политике родителя — узкий lock контейнера назначения (`ContainerDispose`), если требуется.
3. Локально переносит сущность в целевой контейнер (один экземпляр на uid).
4. Ставит в очередь операцию `PickUpEntity` с `resetGeneration`.
5. Отправляет на бекенд при отсутствии другого in-flight по этому uid.

Ok → `Unlock`. Fail → откат в предыдущее расположение (например обратно на землю / в машину) и `Unlock`.

---

### enqueueMoveEntity

**Сигнатура:**

```text
enqueueMoveEntity(string entityUid, string targetContainerUid, string characterUid, string slot?): void
```

**Параметры:**

- `entityUid` — перемещаемая сущность;
- `targetContainerUid` — контейнер назначения;
- `characterUid` — персонаж, инициировавший перемещение;
- `slot` — слот назначения, если нужен.

**Описание:**

1. Проверяет EntityLockRegistry и доменные правила (вместимость, владение контейнером и т.п.).
2. `Lock(entityUid, reason: Move, scope: InventoryOps, ownerService: EntityManager)`.
3. Если цель — рюкзак / носимый контейнер и есть риск dispose родителя до ответа бекенда — дополнительно `Lock(…, scope: ContainerDispose)` по политике вложенности (рюкзак — да; машину для езды — нет).
4. Локально обновляет связи контейнер/слот.
5. Ставит в очередь `MoveEntity` с `resetGeneration` и отправляет при возможности.

Ok → снять все lock’и этой операции через EntityLockRegistry. Fail → откат связей и Unlock.

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
- [[EntityManager.Operations.md]] — логика операций и очередь
- [[EntityManager.DupeAnalyzer.md]] — анализатор дюпов, hard-reset, игнор операций
- [[TradeManager.md]] — sell и Lock на время продажи
- [[CharacterStateManager (черновик) 3.md]] — состояние персонажа; инвентарь только как проекция
- [[../CommandBus/CommandBus.md]] — доставка команд с бекенда
- [[../CommandBus/Commands.md]] — список команд игрового сервера
