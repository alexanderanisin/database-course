# mongoDB, Александр Анисин

## Подготовка окружения

1.  **Запуск контейнеров:** использовал `docker-compose` для поднятия MongoDB и Mongo Express.
2.  **Импорт данных:** загрузил исходные данные из файла `products.json` в коллекцию `products` базы данных `shop`.

```bash
docker cp products.json mng:/products.json
docker exec mng mongoimport --db shop --collection products --file /products.json --jsonArray -u root -p password123 --authenticationDatabase admin
```

## Часть 1. Базовые операции

### 1. Добавление данных (Create)
Добавил в базу новый товар "USB-C Hub" от производителя "TechCorp".

```javascript
db.products.insertOne({
    name: "USB-C Hub",
    brand: "TechCorp",
    price: 45,
    category: "accessories",
    in_stock: true,
    stock_count: 300
})
```

### 2. Поиск данных (Read)

**Поиск по бренду:**
Нашел все товары производства "TechCorp".
```javascript
db.products.find({ brand: "TechCorp" })
```

**Фильтрация по цене:**
Выбрал товары с ценой не более 100.
```javascript
db.products.find({ price: { $lte: 100 } })
```

**Сложный запрос:**
Выбрал товары из категории "peripherals", которых нет в наличии. Вывел только названия и бренды, скрыв остальные поля.
```javascript
db.products.find(
    { category: "peripherals", in_stock: false },
    { name: 1, brand: 1, _id: 0 }
)
```

### 3. Обновление данных

**Массовое обновление (Ребрендинг):**
Обновил название бренда у всех товаров "GamerGear" на "GamerPro".
```javascript
db.products.updateMany(
    { brand: "GamerGear" },
    { $set: { brand: "GamerPro" } }
)
```

**Изменение числового поля:**
Уменьшил цену на "Laptop Pro" на 100 единиц, используя оператор инкремента с отрицательным значением.
```javascript
db.products.updateOne(
    { name: "Laptop Pro" },
    { $inc: { price: -100 } }
)
```

### 4. Удаление данных
Удалил товар "Webcam HD" из базы.
```javascript
db.products.deleteOne({ name: "Webcam HD" })
```

## Часть 2. Продвинутые операции

### 1. Сложная фильтрация
Нашел товары, соответствующие двум условиям одновременно: бренд **не** "TechCorp" **и** цена в диапазоне от 70 до 500 включительно.
```javascript
db.products.find({
    brand: { $ne: "TechCorp" },
    price: { $gte: 70, $lte: 500 }
})
```

### 2. Работа с массивами
Добавил новый тег "best-seller" в массив `tags` для всех товаров, количество которых на складе превышает 100 штук. Оператор `$push` добавляет значение в массив.
```javascript
db.products.updateMany(
    { stock_count: { $gt: 100 } },
    { $push: { tags: "best-seller" } }
)
```

### 3. Условное обновление (Upsert)
Попытался обновить товар "8K Monitor". Так как такого товара не существовало, благодаря опции `{ upsert: true }` MongoDB создал новый документ с указанными полями.
```javascript
db.products.updateOne(
    { name: "8K Monitor" },
    { $set: { price: 2000, brand: "ViewSonic" } },
    { upsert: true }
)
```

### 4. Очистка коллекции
Для демонстрации массового удаления создал временную коллекцию `test_delete`, заполнил её тестовыми данными, а затем удалил из неё все документы одной командой.
```javascript
// Создание тестовых данных
db.test_delete.insertMany([
    { item: 1 }, { item: 2 }, { item: 3 }, { item: 4 }, { item: 5 }
])

// Полная очистка коллекции
db.test_delete.deleteMany({})
```
