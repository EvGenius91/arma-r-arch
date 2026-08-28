# GameManager

Назад: [[Architecture.md]] · [[GameManager.Entities.md]]

GameManager хранит жизненный цикл игровой сессии на бекенде. Игровой сервер при старте мира сообщает `serverSessionUid`; бекенд регистрирует [[GameManager.Entities.md#ServerSession|ServerSession]]. Одновременно активна только одна сессия: запись с `stoppedAt = null`.

Доменный контракт бекенда: `GameService` (`Contracts/GameService/`). RPC: `game@gameStarted` ([[../api/http api.md#gamestarted]]).

Копирование `serverSessionUid` на сущности реестра ([[EntityManager.Entities.md#EntityItem]]) — отдельный шаг, здесь не выполняется.

## Структура документации

- [[GameManager.Entities.md]] — `ServerSession`, `GameStartedResult`

## Методы

### gameStarted

**Сигнатура**
gameStarted(string serverSessionUid): [[GameManager.Entities.md#GameStartedResult]]

**Аргументы**
- `serverSessionUid: string` — идентификатор сессии игрового сервера (любая строка). Задаётся игрой, бекенд не генерирует.

**Описание**
Игровой сервер вызывает метод при старте. Бекенд закрывает все другие активные сессии (`stoppedAt = now()`) и регистрирует `ServerSession`: `startedAt = now()`, `stoppedAt = null`. Активна только одна сессия.

Повтор с тем же uid, если эта сессия уже активна, идемпотентен: существующая запись не меняется, `startedAt` не перезаписывается, себя метод не закрывает.

Доменных отказов нет. Невалидные params — отказ на уровне запроса (`InvalidParams`), не `failReason` сервиса.

После регистрации сессии бекенд ставит в очередь CommandBus команду `SpawnEntity` для каждой сущности в мире без владельца-персонажа и без родительского контейнера. Вложенные предметы отдельными командами не ставятся: игра подтягивает их через `entity@findEntitiesByUidList` при спавне контейнера. Предметы персонажа материализуются через `loadInventory`.

**Результат**
- При успехе возвращает `status = Ok`, `failReason = null`.

**Ошибки / причины отказа**
- Доменных отказов нет.

**Связано**
[[GameManager.Entities.md#ServerSession]], [[GameManager.Entities.md#GameStartedResult]], [[../CommandBus/Commands.md]]
