

===== docker-compose.yaml

services:
  mongo:
    image: mongo:latest
    container_name: mng
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password123
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  mongo-express:
    image: mongo-express:latest
    container_name: mnge
    restart: always
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: password123
      ME_CONFIG_MONGODB_SERVER: mongo

volumes:
  mongo-data:
  
========
`services.mongo`: Запускает контейнер с последней версией MongoDB.
`services.mongo-express`: Запускает веб-интерфейс для управления базой.
`environment`: Задает логин и пароль для root-пользователя. 
**Важно:** `ME_CONFIG_MONGODB_SERVER: mongo` указывает `mongo-express`, что искать базу данных нужно по имени сервиса `mongo`. Docker Compose обеспечивает сетевое взаимодействие между ними.
`ports`: "Пробрасывает" порты из контейнеров на вашу машину. `27017` — стандартный порт MongoDB, `8081` — для веб-интерфейса.
`volumes`: Создает именованный том `mongo-data`, чтобы данные не удалялись при перезапуске контейнера.

В той же директории, где лежит файл, выполните в терминале команду:
docker-compose up -d

Выполните команду `docker ps`. Вы должны увидеть два запущенных контейнера: `mng` и `mnge`.
Откройте браузер и перейдите по адресу `http://<IP_вашей_ВМ>:8081`. Вы увидите веб-интерфейс Mongo Express.
Для подключения к MongoDB через Shell, выполните команду:
docker exec -it mng mongosh -u root -p password123
Вы попадете в интерактивную консоль MongoDB.

**Файл `products.json`**

[
  {"name": "Laptop Pro", "brand": "TechCorp", "price": 1500, "category": "electronics", "tags": ["powerful", "professional", "15-inch"], "in_stock": true, "stock_count": 50},
  {"name": "Gaming Mouse", "brand": "GamerGear", "price": 80, "category": "peripherals", "tags": ["rgb", "fast", "ergonomic"], "in_stock": true, "stock_count": 200},
  {"name": "Wireless Keyboard", "brand": "TechCorp", "price": 120, "category": "peripherals", "tags": ["bluetooth", "mechanical", "compact"], "in_stock": false, "stock_count": 0},
  {"name": "4K Monitor", "brand": "ViewSonic", "price": 450, "category": "electronics", "tags": ["4k", "27-inch", "ips"], "in_stock": true, "stock_count": 75},
  {"name": "Webcam HD", "brand": "GamerGear", "price": 60, "category": "peripherals", "tags": ["1080p", "streaming"], "in_stock": true, "stock_count": 150},
  {"name": "Smartphone X", "brand": "TechCorp", "price": 900, "category": "electronics", "tags": ["5g", "oled", "camera"], "in_stock": false, "stock_count": 0}
]

**Инструкция по импорту:**
Для импорта этого файла нужно скопировать его в контейнер, а затем выполнить `mongoimport`.
docker cp products.json mng:/products.json
docker exec -it mng mongoimport --db shop --collection products --file /products.json --jsonArray -u root -p password123 --authenticationDatabase admin

---

### Задания 

**Подготовка:**
1.  Создайте базу данных `shop`.
2.  Используя `insertMany`, вставьте в коллекцию `products` данные из файла `products.json`. Можете просто скопировать и вставить массив в команду `db.products.insertMany(...)` в Shell.

**Часть 1 (Средний уровень):**

1.  **Create:** Добавьте новый товар: "USB-C Hub" от "TechCorp" ценой 45, категория "accessories", в наличии, 300 штук на складе.
2.  **Read:**
    *   Найдите все товары бренда "TechCorp".
    *   Найдите все товары, цена которых **меньше или равна** 100.
    *   Найдите все товары из категории "peripherals", которых **нет в наличии** (`in_stock: false`). Верните только их названия и бренды (`name`, `brand`).
3.  **Update:**
    *   Бренд "GamerGear" провел ребрендинг и теперь называется "GamerPro". Обновите **все** товары этого бренда.
    *   На "Laptop Pro" объявлена распродажа. Уменьшите его цену на 100 с помощью оператора `$inc`.
4.  **Delete:**
    *   "Webcam HD" оказался бракованным. Удалите этот товар из базы.

**Часть 2 (Продвинутый уровень):**

1.  **Read (сложный фильтр):** Найдите все товары, которые **не** относятся к бренду "TechCorp" **и** цена которых находится в диапазоне от 70 до 500 (включительно).
2.  **Update (работа с массивами):** Добавьте тег "best-seller" всем товарам, у которых на складе (`stock_count`) более 100 единиц. Используйте `updateMany` и оператор `$push`.
3.  **Update (условное обновление):** Найдите "4K Monitor". Используя `updateOne` с опцией `{ upsert: true }`, попробуйте обновить несуществующий товар, например, "8K Monitor". Убедитесь, что новый документ был создан.
    *   `db.products.updateOne({ name: "8K Monitor" }, { $set: { price: 2000, brand: "ViewSonic" } }, { upsert: true })`
4.  **Delete (с осторожностью):** Создайте новую коллекцию `test_delete` и вставьте в нее 5 любых документов. Затем удалите из нее все документы одной командой.
