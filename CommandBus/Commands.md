### spawnNewVehicleToNearSlot
**Сигнатура:**
spawnNewVehicleToNearSlot(string prefabName, string vehicleUid, string characterUid): SpawnNewVehicleToNearSlotResult;
**Описание:**
заспавнить технику на ближайшей слоте и добавить uid к технике.
**Возможные причины невыполнения:**
Нет свободных слотов
Рядом нет слота

### addNewEntityToCharacterInventory
**Сигнатура:**
addNewEntityToCharacterInventory(string prefabName, string entityUid, string characterUid): AddNewEntityToCharacterInventory;
**Описание:**
добавить элемент в инвентарь персонажа и добавить uid к элементу
**Возможные причины невыполнения:**
Нет места в инвентаре

removeVehicle(string vehicleUid) - удалить технику, отметить в реестре технику как удаленную
Возможные причины невыполнения:
Техника с заданным uid не найдена

removeEntity(string entityUid) - удалить элемент, отметить в реестре как удаленный
Возможные причины невыполнения:
Элемент с заданным uid не найден