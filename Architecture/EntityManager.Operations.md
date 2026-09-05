# Операции EntityManager (игра → бекенд)

Назад: [[EntityManager.md]] · [[Architecture.md]]

Логика обработки операций расположения сущностей: публичный API (регистрация уже случившегося в мире изменения), исходящая очередь, flush (таймер / barrier), блокировки через [[EntityLockRegistry.md|EntityLockRegistry]] на время in-flight, реакция на ответ бекенда. Дюпы и hard-reset — отдельно: [[EntityManager.DupeAnalyzer.md]].

## Главные правила

1. **Точка входа** — EntityManager (`enqueuePickupEntity`, `enqueueDropEntity`, `enqueueMoveEntity`, `enqueueEntityTransformChanged`, `enqueueEntityHitZoneChanged`, …). Инвентарь / мир / UI не ходят на бекенд напрямую и не через CharacterStateManager.
2. **Один `entityUid` — один экземпляр** в мире (перенос объекта, не клон при RPC).
3. Ops складываются в **исходящую очередь**; на бекенд уходят **пачкой** по таймеру (~1 с) или по **barrier**.
4. В пачке ops **обязательно сохраняют порядок** (особенно цепочка по одному uid).
5. **Lock** в EntityLockRegistry — только пока по uid есть блокирующая **in-flight** операция (пачка ушла, ответа ещё нет). Пока ops лишь в очереди — uid не locked. `EntityTransformChanged` и `EntityHitZoneChanged` являются неблокирующими исключениями и не создают lock даже в in-flight.
6. Блокировка — **по сущности** (и узко по родителю-контейнеру на время in-flight), не глобальная очередь на весь мир.
7. Бекенд — источник истины; отказ блокирующей операции → откат расположения в мире. Для `EntityTransformChanged` и `EntityHitZoneChanged` физического отката нет: следующий снимок повторно выравнивает бекенд. Мутации с бекенда в мир — через CommandBus ([[BackendGameMutation.md]]), не из Trade напрямую.
8. Каждая операция несёт `resetGeneration` сущности ([[EntityManager.DupeAnalyzer.md]]).
9. Trade на время sell пишет в тот же реестр (`SellPending`) — это отдельный lock продажи, не очередь EM.

---

## Модель очереди и отправки (итог)

```text
enqueue* → проверки → зафиксировать расположение в реестре (мир уже изменился для PickUp) → запись в очередь (без Lock)
         → flush: ~1 с  ИЛИ  barrier
         → Lock затронутых uid для блокирующих ops → entity@applyOperations (порядок сохранён) → in-flight
         → Ok / Fail → Unlock блокирующих ops → при Fail откат только блокирующих изменений; при необходимости следующая пачка по uid
```

| Фаза | Поведение по uid |
|------|------------------|
| В очереди, ещё не ушло | расположение уже в мире сервера (для PickUp — сдвинул инвентарь); lock нет; новые ops (в т.ч. другого игрока) дописываются в очередь **в порядке поступления** |
| Пачка ушла, ждём ответ (in-flight) | Для блокирующих ops: `EntityLockRegistry.Lock` — нельзя drop / pick / move / sell этого uid. `EntityTransformChanged` и `EntityHitZoneChanged` lock не создают |
| Ok | `Unlock`, мир совпал (или дотянуть мелочи), EventBus |
| Fail | Для блокирующих ops — откат расположения и `Unlock`; для `EntityTransformChanged` и `EntityHitZoneChanged` — журналирование без физического отката |

### Когда flush

| Триггер | Что уходит |
|---------|------------|
| Таймер ~1 с | накопленные pending ops (по политике EM — все готовые uid или с лимитом размера пачки) |
| Barrier | принудительный flush и ожидание settled для нужного набора uid |

Примеры barrier:

- игрок открыл магазин → все предметы **его инвентаря** не должны быть in-flight (и pending по ним нужно сбросить и дождаться ответа), прежде чем `getInventoryForSell` / UI;
- старт `sell` / иные точки, где снимок инвентаря должен совпасть с бекендом.

Barrier **скоупается** по смыслу события: открытие магазина игроком 2 не обязано flush’ить дроп топора игрока 1, если топор не в инвентаре игрока 2.

### Пачка и группировка по uid

- **Одна сетевая пачка ≠ один uid.** В одном flush могут уйти ops по нескольким uid.
- **Сериализация — по uid:** порядок ops одного uid в пачке и на бекенде сохраняется. Lock и запрет параллельных блокирующих операций применяются к inventory/location ops; `EntityTransformChanged` и `EntityHitZoneChanged` перед flush объединяются до последнего снимка и lock не используют.
- На бекенде пачка **не** обязана быть одной атомарной транзакцией на все uid («банка + топор + пила»). Успех / отказ — **по uid** (или по отдельным ops); админ-`DeleteEntity` одной банки не должен ронять остальные uid пачки.

---

## Публичный API

Системы мира вызывают `enqueue*` — **одна операция: мир уже изменил расположение (для PickUp), EntityManager регистрирует связи и ставит в очередь**. Снаружи не собирают пачки и не знают про HTTP.

```text
enqueuePickupEntity(entityUid, targetContainerUid?, characterUid, storageType, slot?)
enqueueDropEntity(entityUid, position, characterUid, storageType)
enqueueMoveEntity(entityUid, targetContainerUid?, characterUid, storageType, slot?)
enqueueEntityTransformChanged(entityUid, position, angle)
enqueueEntityHitZoneChanged(entityUid, hitZoneName, healthScaled)
// далее по тому же шаблону:
// enqueueEquipItem / enqueueTransferEntity / …
```

Сигнатуры и шаги — в [[EntityManager.md#enqueuedropentity|EntityManager.md]] (методы). In-flight и момент `Lock`/`Unlock` — этот документ и [[EntityLockRegistry.md|EntityLockRegistry]].

Trade перед `sell`: при необходимости barrier/flush по uid; затем `IsLocked` / `Lock` с `SellPending`. См. [[EntityLockRegistry.md]].

Опционально EM может дать явный `flushPendingOperations(characterUid?)` / `ensureSettled(entityUids|characterUid)` для barrier из UI / Trade — без сборки пачки снаружи.

---

## EntityTransformChanged

`EntityTransformChanged` фиксирует актуальную позицию и угол сущности, которая уже находится в мире. Наблюдатель игрового мира вызывает `enqueueEntityTransformChanged(entityUid, position, angle)` при изменении хотя бы одного из этих значений.

Политика очереди отличается от операций смены владения / контейнера:

1. Для одного `entityUid` в pending-очереди хранится только последний неотправленный transform: новая операция заменяет предыдущую.
2. Coalescing не пересекает операцию, меняющую тип расположения. После `DropEntity` transform остаётся после дропа; при нескольких transform после него сохраняется только последний.
3. Если сущность подобрана в контейнер / слот, pending `EntityTransformChanged` для её прежнего положения в мире удаляется перед `PickUpEntity`.
4. Операция уходит через общий `entity@applyOperations` и содержит `resetGeneration`, но не ставит `EntityLockRegistry` lock, не блокирует управление / движение сущности и не участвует в `InventoryOps`. Пока transform этого uid in-flight, новые снимки продолжают объединяться в pending; второй запрос по тому же uid отправляется только после ответа на первый, чтобы ответы не переупорядочили координаты.
5. При `Fail` игровой сервер не возвращает сущность на старые координаты. Ошибка журналируется; следующий transform-снимок формирует новую попытку синхронизации.
6. На бекенде операция допустима только для сущности в мире. Контракт: `characterUid = null`, `payload = { position, angle }`, без `storageType`.

```text
Transform A → Transform B → Transform C → flush
pending:                         [EntityTransformChanged C]

Drop → Transform A → Transform B → flush
pending: [DropEntity, EntityTransformChanged B]

Transform A → PickUp → flush
pending: [PickUpEntity]
```

---

## EntityHitZoneChanged

`EntityHitZoneChanged` фиксирует разреженный оверлей HitZone экземпляра. Наблюдатель игрового мира вызывает `enqueueEntityHitZoneChanged(entityUid, hitZoneName, healthScaled)` при изменении здоровья зоны.

Политика очереди совпадает с [[#EntityTransformChanged]] по lock и Fail:

1. Для одного `entityUid` в pending хранится одна ops: map `name → healthScaled`. Новый вызов сливается в этот оверлей (последнее значение по имени побеждает).
2. Зона с `healthScaled == 1` из оверлея выкидывается. Пустой `hitZones` после слияния — сущность снова как из префаба (полный ремонт); такая ops всё равно уходит, чтобы стереть оверлей на бекенде.
3. Coalescing HitZone не выкидывает и не переставляет inventory/location ops того же uid. Pending оверлей живёт независимо от `PickUpEntity` / `DropEntity` / `MoveEntity`.
4. Операция уходит через общий `entity@applyOperations` и содержит `resetGeneration`, но не ставит `EntityLockRegistry` lock, не блокирует управление / движение / инвентарь и не участвует в `InventoryOps`. Пока HitZone этого uid in-flight, новые снимки продолжают объединяться в pending; второй запрос по тому же uid — только после ответа на первый.
5. При `Fail` игровой сервер не откатывает HP зон в мире. Ошибка журналируется; следующий снимок формирует новую попытку.
6. Контракт: `characterUid = null`, `payload = { hitZones }`, без `storageType`. Бекенд **заменяет** сохранённый оверлей целиком списком из payload (не патч одной зоны).

```text
Wheel_1=0.2 → Engine=0.4 → Wheel_1=0.0 → flush
pending: [EntityHitZoneChanged { Wheel_1: 0.0, Engine: 0.4 }]

Engine=0.0 → Engine=1.0 → flush
pending: [EntityHitZoneChanged { hitZones: [] }]
```

---

## Жизненный цикл блокирующей операции

```text
1. Вызов enqueueDropEntity / enqueuePickupEntity / enqueueMoveEntity / …
2. Проверки: сущность есть; если uid уже in-flight (IsLocked от EM) — отказ
   (не ставить новые игровые ops, пока летит пачка); доменные правила игры
3. Для PickUp: мир уже перенёс IEntity (инвентарь / хук); EM только обновляет связи
   в локальном реестре и снимок прежнего расположения для отката. EM не двигает предмет повторно.
   (Drop / Move — источник переноса в мире у вызывающей системы; EM регистрирует и ставит в очередь.)
4. Поставить операцию в исходящую очередь (с resetGeneration) — без Lock
5. Flush (таймер ~1 с или barrier):
   Lock затронутых uid (+ родитель ContainerDispose по политике)
   отправить пачку через `entity@applyOperations` с сохранением порядка
6. Ответ по uid / ops:
   Ok  → Unlock, зафиксировать связи, событие в Local EventBus
   Fail → откат расположения в мире, Unlock, событие / сигнал UI
7. Снять из in-flight; при наличии хвоста очереди — следующий flush по правилам
```

```mermaid
sequenceDiagram
    participant World as Игровой_мир
    participant EM as EntityManager
    participant Lock as EntityLockRegistry
    participant Q as Очередь
    participant BE as Бекенд

    World->>World: подбор_в_инвентарь
    World->>EM: enqueuePickupEntity_банка
    EM->>EM: IsLocked_нет
    EM->>EM: реестр_связей_снимок
    EM->>Q: PickUp_в_очередь
    Note over Q: ждём_таймер_или_barrier
    EM->>Lock: Lock_банка
    EM->>BE: entity@applyOperations
    alt Ok
        BE-->>EM: Ok
        EM->>Lock: Unlock_банка
    else Fail
        BE-->>EM: Fail
        EM->>EM: откат_в_мире
        EM->>Lock: Unlock_банка
    end
```

---

## Очередь и in-flight

Пока ops по `entityUid` **только в очереди** (ещё не ушли):

- второй игрок **может** подобрать предмет, если локальный мир сервера уже показывает его на земле после дропа;
- обе ops попадают в одну упорядоченную очередь uid: `[Drop P1, PickUp P2]` и могут уйти в одной пачке.

Пока по `entityUid` есть **in-flight** (пачка ушла, ответа нет):

- игровые операции с этим uid **блокируются** (`IsLocked`) — нельзя drop / pick / move / sell;
- параллельного второго запроса на бекенд по этому uid нет;
- новые попытки — отказ, пока нет Unlock.

Операции по **другим** uid не ждут эту банку (нет глобальной блокировки мира), кроме общего тика flush / своего barrier.

---

## Родительский контейнер

| Ситуация | Политика |
|----------|----------|
| Положить банку в рюкзак | на время in-flight банки — lock банки; рюкзак — как минимум `ContainerDispose` |
| Положить банку в машину | lock банки на in-flight; машину **не** блокировать для вождения |
| In-flight по содержимому, затем dispose контейнера | dispose контейнера нельзя, пока содержимое in-flight; либо в очередь после Unlock |

---

## Продажа и передача

### Передача / дроп / подбор

`enqueue*` → регистрация расположения + очередь. `Lock` — при уходе пачки в in-flight. Trade / другие игроки видят `IsLocked` только в этом окне (и при `SellPending`).

### Продажа (Trade)

1. При необходимости barrier: flush + дождаться, пока продаваемые uid не in-flight.  
2. Trade → EntityLockRegistry: `IsLocked(uid)`? если да — sell не слать.  
3. Если нет — `Lock(uid, SellPending, InventoryOps, TradeManager)` (игрок не сможет `enqueueDropEntity`).  
4. Успех sell на бекенде → удаление в мире через CommandBus `DeleteEntity` ([[BackendGameMutation.md]]).  
5. `DeleteEntity` идемпотентен и выполняется **даже при lock**; после `DeleteEntity` — `Unlock` и отмена pending ops EntityManager по uid.  

Подробный сценарий: [[EntityLockRegistry.md#сценарий-продажа]].

---

## Ответ бекенда и CommandBus

| Событие | Действие EntityManager |
|---------|-------------------------|
| Ok операции расположения | Unlock, мир уже совпал (или дотянуть мелочи), EventBus |
| Fail блокирующей операции | откат расположения в мире, Unlock |
| Fail `EntityTransformChanged` | журналировать; не откатывать физическое перемещение; следующий снимок повторит синхронизацию |
| Fail `EntityHitZoneChanged` | журналировать; не откатывать HP HitZone в мире; следующий снимок повторит синхронизацию |
| Fail / StaleAfterReset ([[EntityManager.DupeAnalyzer.md]]) | не двигать мир по этой ops; мир уже/будет выровнен hard-reset или сверкой |
| CommandBus `DeleteEntity` | удалить экземпляр где угодно (инвентарь / земля / «у другого»), Unlock, очистить очередь ops по uid |
| CommandBus `SpawnEntity` | создать сущности в мире по команде (`enqueueCommand` → `reportCommands`) |

EntityManager **не** ждёт второй вызов CharacterStateManager для удаления предмета. CharacterStateManager при необходимости обновляет проекцию/версию по событию или `CharacterStateChanged`.

---

## Поля операции на бекенд (минимум)

```text
entityUid
type                    // PickUpEntity | DropEntity | MoveEntity | EntityTransformChanged | EntityHitZoneChanged | …
resetGeneration         // поколение сущности на момент отправки
characterUid            // инициатор; null для EntityTransformChanged и EntityHitZoneChanged
payload                 // поля по type; для EntityTransformChanged: position + angle; для EntityHitZoneChanged: hitZones
```

**RPC:** `entity@applyOperations` — пачка `operations[]` при flush. Полный контракт JSON-RPC: [[EntityManager.HttpMethods.md]], транспорт: [[../api/http api.md]].

Бекенд применяет ops **с сохранением порядка** в рамках uid. Второй одновременный успешный take одной банки на бекенде не допускается (второй → Fail) — на игровом сервере это закрывается очередью + lock на in-flight.

---

## Кейсы

### Кейс A — дроп и подбор вторым игроком (одна пачка)

1. Игрок 1 дропает банку → банка на земле в мире, `Drop` в очереди (lock ещё нет).  
2. Игрок 2 подбирает банку → банка уже в инвентаре P2 в мире, EM ставит `PickUp` в очередь uid: `[Drop P1, PickUp P2]`.  
3. Flush (таймер или barrier) → `Lock(банка)` → пачка в том же порядке → ответ → `Unlock`.  
4. Рассинхрона нет: общий EM на игровом сервере, порядок на бекенде тот же.

Пока пачка по банке **in-flight**, подобрать / дропнуть / продать эту банку нельзя.

### Кейс B — barrier магазина (скоуп инвентаря)

1. Игрок 1 бросил банку и топор → в очереди `Drop(банка)`, `Drop(топор)`.  
2. Игрок 2 подобрал банку → очередь банки: `[Drop P1, PickUp P2]`; топор по-прежнему `[Drop P1]` на земле.  
3. Игрок 2 открывает магазин; в инвентаре банка и пила (по пиле ops не было).  
4. Barrier магазина P2: flush + settled **только по uid инвентаря P2** (банка). На бекенд уходит очередь банки; **ops топора остаются в локальной очереди** до таймера ~1 с или своего barrier.  
5. Пила не шлётся — очередь пуста.

### Кейс C — передача и продажа

1. Пока transfer по uid in-flight — `IsLocked`, продажа локально запрещена.  
2. `SellPending` на время sell закрывает `enqueueDropEntity` / передачу.

### Кейс D — админ удалил банку, в пачке ещё топор и пила

1. Команда `DeleteEntity(банка)` → банка исчезает, Unlock банки, ops по банке отменяются.  
2. Операции топора и пилы (другие uid) обрабатываются отдельно → Ok/Fail сами по себе.

### Кейс E — банка в рюкзак, затем выброс рюкзака

Пока банка in-flight, выброс рюкзака локально запрещён (`ContainerDispose`) или ждёт Unlock банки.

---

## Граница с CharacterStateManager

| EntityManager | CharacterStateManager | EntityLockRegistry |
|---------------|------------------------|-------------------|
| содержимое контейнеров, связи uid, позиция в мире, оверлей HitZone экземпляра | id надетых контейнеров, HP / витальность / поза персонажа | lock uid на время in-flight EM и SellPending Trade |
| отправка item operations (очередь + flush) | не шлёт item operations | не хранит расположение предметов |

---

## Анти-паттерны

1. Глобальная блокировка всех операций мира из‑за одной банки.  
2. Полный lock машины (нельзя ехать), когда банка кладётся в инвентарь техники.  
3. Клонирование объекта при «подборе» через RPC (два экземпляра одного uid).  
4. Sell без `EntityLockRegistry.Lock(SellPending)` — игрок успевает выбросить предмет.  
5. Отдельные lock внутри Trade и EntityManager вместо общего сервиса.  
6. Отказ выполнить CommandBus `DeleteEntity` из‑за lock.  
7. Атомарная пачка из многих uid как единственный способ commit на бекенде (хрупко при админ-delete одного предмета).  
8. Дублировать дерево содержимого рюкзака в CharacterStateManager.  
9. `Lock` на всё время нахождения ops в очереди (до flush) — мешает легитимной цепочке Drop→PickUp в одной пачке.  
10. Параллельный in-flight по одному uid / отправка пачки без сохранения порядка.  
11. Barrier магазина, который flush’ит весь мир вместо uid инвентаря открывшего игрока.

---

## Связанные документы

- [[EntityManager.md]] — реестр и обзор
- [[EntityManager.HttpMethods.md]] — JSON-RPC `entity@applyOperations`
- [[EntityLockRegistry.md]] — общий сервис блокировок
- [[EntityManager.DupeAnalyzer.md]] — дюпы, hard-reset, `resetGeneration`
- [[TradeManager.md]] — sell и SellPending
- [[BackendGameMutation.md]] — бекенд → CommandBus → мир
- [[CharacterStateManager (черновик) 3.md]] — состояние персонажа без дерева лута
- [[../api/http api.md]] — транспорт HTTP JSON-RPC
- [[../CommandBus/CommandBus.md]] — доставка команд
- [[../CommandBus/Commands.md]] — каталог команд
