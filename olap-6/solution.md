## Задача 1

Схема звезды/снежинки под описанную задачу. Продукты хранятся с привязкой к бренду, магазины через цепочку провинция → страна, время — отдельная таблица со всеми нужными атрибутами.

### Реляционная схема

```sql
Brand(brand_id PK, brand_name)
Product(product_id PK, product_name, brand_id FK → Brand)

Country(country_id PK, country_name)
Province(province_id PK, province_name, country_id FK → Country)
Store(store_id PK, store_name, province_id FK → Province)

Time(date_id PK, date, day_of_week, month, quarter, semester, year)

Sales(product_id FK, store_id FK, date_id FK, amount)
  PRIMARY KEY (product_id, store_id, date_id)
```

### SQL:1999 — CUBE

```sql
SELECT
    p.product_id, p.brand_id,
    s.store_id, s.province_id, s.country_id,
    t.year, t.semester, t.quarter, t.month, t.day_of_week,
    SUM(f.amount) AS total_sales,
    COUNT(*)      AS cnt
    -- среднее = total_sales / cnt, считать на уровне запроса
FROM Sales f
JOIN Product  p  ON f.product_id  = p.product_id
JOIN Store    s  ON f.store_id    = s.store_id
JOIN Province pr ON s.province_id = pr.province_id
JOIN Country  c  ON pr.country_id = c.country_id
JOIN Time     t  ON f.date_id     = t.date_id
GROUP BY CUBE(
    p.product_id, p.brand_id,
    s.store_id, s.province_id, s.country_id,
    t.year, t.semester, t.quarter, t.month, t.day_of_week
);
```

NULL в атрибуте измерения = агрегация по нему (ALL).

### Почему AVG нельзя хранить напрямую

`AVG` не раскладывается по уровням: среднее по всем продажам за год ≠ среднее ежемесячных средних. Хранить надо `SUM` и `COUNT`, среднее считать как `SUM / COUNT` при чтении.

### Иерархии и проблема с CUBE

Чистый `CUBE` даёт бессмысленные комбинации — например `(product_id = X, brand_id = NULL)`, то есть конкретный продукт без бренда. По иерархии такого быть не может.

Чтобы получать только осмысленные срезы, вместо CUBE используем ROLLUP по каждому измерению:

```sql
GROUP BY
    ROLLUP(p.brand_id, p.product_id),
    ROLLUP(c.country_id, pr.province_id, s.store_id),
    GROUPING SETS(
        (t.year, t.month),
        (t.year, t.quarter),
        (t.year, t.semester),
        (t.year, t.day_of_week),
        (t.year),
        ()
    )
```

`ROLLUP(brand_id, product_id)` строит комбинации справа налево: (бренд, продукт) → (бренд) → (). Комбинация (NULL, продукт) никогда не появится.

## Задача 2

`Sales(Product, Month, Store, Amount)` — 5 продуктов, 12 месяцев, 3 магазина.

### (a) Плотный случай

**(i)** Каждая тройка (продукт, месяц, магазин) даёт ровно один кортеж:

$$5 \times 12 \times 3 = \mathbf{180}$$

**(ii)** Датакуб включает все $2^3 = 8$ агрегаций. По каждому измерению добавляется один уровень ALL:

| Представление           | Размер           |
| ----------------------- | ---------------- |
| (Product, Month, Store) | 5 × 12 × 3 = 180 |
| (Product, Month, ALL)   | 5 × 12 = 60      |
| (Product, ALL, Store)   | 5 × 3 = 15       |
| (ALL, Month, Store)     | 12 × 3 = 36      |
| (Product, ALL, ALL)     | 5                |
| (ALL, Month, ALL)       | 12               |
| (ALL, ALL, Store)       | 3                |
| (ALL, ALL, ALL)         | 1                |
| **Итого**               | **312**          |

### (b) Разреженный случай

Исходные кортежи:

| Product | Month | Store |
| ------- | ----- | ----- |
| P1      | Jan   | S1    |
| P1      | Jan   | S2    |
| P2      | Feb   | S2    |
| P2      | Feb   | S3    |
| P3      | Jan   | S1    |
| P3      | Feb   | S1    |
| P4      | Feb   | S1    |
| P5      | Jan   | S3    |

Проходим по всем 8 представлениям и считаем непустые ячейки:

| Представление           | Непустых ячеек                                            |
| ----------------------- | --------------------------------------------------------- |
| (Product, Month, Store) | 8                                                         |
| (Product, Month)        | {P1/Jan, P2/Feb, P3/Jan, P3/Feb, P4/Feb, P5/Jan} = **6**  |
| (Product, Store)        | {P1/S1, P1/S2, P2/S2, P2/S3, P3/S1, P4/S1, P5/S3} = **7** |
| (Month, Store)          | {Jan/S1, Jan/S2, Feb/S2, Feb/S3, Feb/S1, Jan/S3} = **6**  |
| (Product)               | P1, P2, P3, P4, P5 = **5**                                |
| (Month)                 | Jan, Feb = **2**                                          |
| (Store)                 | S1, S2, S3 = **3**                                        |
| (ALL)                   | **1**                                                     |
| **Итого**               | **38**                                                    |

## Задача 3

Датакуб: измерения `Product` и `Date`, мера `Total Sales`.

Иерархия на `Date`:

```
      year
        |
  month   week
      \   /
       Date
```

100 продуктов, 1095 дней, 36 месяцев, 157 недель, 3 года. Куб плотный.

`month` и `week` несравнимы — недели не делятся на месяцы без остатка, поэтому из одного нельзя получить другое.

Решётка: {Product, ALL} × {Date, month, week, year, ALL} = **10 узлов**.

### (a) Размеры представлений

| Представление    | Размер                   |
| ---------------- | ------------------------ |
| (Product, Date)  | 100 × 1095 = **109 500** |
| (Product, month) | 100 × 36 = **3 600**     |
| (Product, week)  | 100 × 157 = **15 700**   |
| (Product, year)  | 100 × 3 = **300**        |
| (Product, ALL)   | 100                      |
| (ALL, Date)      | 1 095                    |
| (ALL, month)     | 36                       |
| (ALL, week)      | 157                      |
| (ALL, year)      | 3                        |
| (ALL, ALL)       | 1                        |

### (b) Жадный алгоритм — выбор 2 представлений

Базовое отношение `(Product, Date)` размером 109 500 материализовано всегда. Все 10 запросов сейчас отвечаются через него, суммарная стоимость = 10 × 109 500 = **1 095 000**.

Выгода от материализации `v` — сколько экономим суммарно по всем узлам `u`, для которых `v` становится более дешёвым источником. `v` может ответить на `u`, только если `v` не менее детальное по обоим измерениям.

**Раунд 1:**

| Кандидат         | Покрывает узлов | Выгода                              |
| ---------------- | --------------- | ----------------------------------- |
| (Product, month) | 6               | 6 × (109 500 − 3 600) = **635 400** |
| (Product, week)  | 6               | 6 × (109 500 − 15 700) = 562 800    |
| (ALL, Date)      | 5               | 5 × (109 500 − 1 095) = 542 025     |
| (Product, year)  | 4               | 4 × (109 500 − 300) = 436 800       |
| (ALL, month)     | 3               | 3 × (109 500 − 36) = 328 392        |
| (ALL, week)      | 3               | 3 × (109 500 − 157) = 328 029       |
| (ALL, year)      | 2               | 2 × (109 500 − 3) = 218 994         |
| (Product, ALL)   | 2               | 2 × (109 500 − 100) = 218 800       |
| (ALL, ALL)       | 1               | 109 499                             |

Берём **(Product, month)**, выгода **635 400**.

**Раунд 2:** S = {(Product, Date), (Product, month)}.

После первого шага 6 узлов отвечаются через `(Product, month)` с ценой 3 600, остальные 4 — всё ещё через базовое (109 500):

- цена 109 500: `(Product, Date)`, `(Product, week)`, `(ALL, Date)`, `(ALL, week)`
- цена 3 600: `(Product, month)`, `(Product, year)`, `(Product, ALL)`, `(ALL, month)`, `(ALL, year)`, `(ALL, ALL)`

| Кандидат        | Расчёт выгоды                                                                                          | Выгода      |
| --------------- | ------------------------------------------------------------------------------------------------------ | ----------- |
| **(ALL, Date)** | (ALL,Date): −108 405; (ALL,month): −2 505; (ALL,week): −108 405; (ALL,year): −2 505; (ALL,ALL): −2 505 | **224 325** |
| (Product, week) | (Prod,week) и (ALL,week) по 93 800                                                                     | 187 600     |
| (ALL, week)     | (ALL,week): 109 343; (ALL,year): 3 443; (ALL,ALL): 3 443                                               | 116 229     |
| (Product, year) | 4 × 3 300                                                                                              | 13 200      |

Берём **(ALL, Date)**, выгода **224 325**.

### (c) Суммарная выгода

$$635\ 400 + 224\ 325 = \mathbf{859\ 725}$$

Проверка через итоговые стоимости:

| Узел             | Источник         | Стоимость   |
| ---------------- | ---------------- | ----------- |
| (Product, Date)  | (Product, Date)  | 109 500     |
| (Product, week)  | (Product, Date)  | 109 500     |
| (Product, month) | (Product, month) | 3 600       |
| (Product, year)  | (Product, month) | 3 600       |
| (Product, ALL)   | (Product, month) | 3 600       |
| (ALL, Date)      | (ALL, Date)      | 1 095       |
| (ALL, month)     | (ALL, Date)      | 1 095       |
| (ALL, week)      | (ALL, Date)      | 1 095       |
| (ALL, year)      | (ALL, Date)      | 1 095       |
| (ALL, ALL)       | (ALL, Date)      | 1 095       |
| **Итого**        |                  | **235 275** |

$1\ 095\ 000 - 235\ 275 = 859\ 725$

## Задача 4

Надо показать, что граница $(e-1)/e$ точная: для любого $k$ найдётся решётка, где $B_\text{greedy}/B_\text{opt}$ ровно равно $1 - \left(\frac{k-1}{k}\right)^k$.

### Конструкция

Берём большое $N$ и строим решётку:

- один узел $T$ (всегда материализован, размер $N$)
- $k$ узлов $V_1, \ldots, V_k$, каждый размера $\approx 0$, все несравнимы друг с другом

Запросы делим на $k$ непересекающихся групп $G_1, \ldots, G_k$. Узел $V_i$ отвечает ровно на запросы из $G_1 \cup \ldots \cup G_i$, причём $|G_i| = N \cdot \left(\frac{k-1}{k}\right)^{i-1}$ — каждая следующая группа меньше предыдущей в $k/(k-1)$ раз.

### Что делает жадный алгоритм

На первом шаге все $V_i$ выглядят одинаково только если группы равные — но у нас они убывают. Жадный всё равно выбирает $V_1$ первым (наибольшая группа), затем $V_2$, и т.д.

На шаге $j$ добавляется ровно $|G_j| = N(1-1/k)^{j-1}$ новых покрытых запросов. Суммарная выгода:

$$B_\text{greedy}(k) = N \sum_{j=0}^{k-1} \left(1 - \frac{1}{k}\right)^j = Nk\left[1 - \left(\frac{k-1}{k}\right)^k\right]$$

Оптимальный алгоритм покрывает все $kN$ запросов, поэтому $B_\text{opt} = Nk$.

Отношение:

$$\frac{B_\text{greedy}}{B_\text{opt}} = 1 - \left(\frac{k-1}{k}\right)^k$$

При $k \to \infty$ это стремится к $1 - 1/e = (e-1)/e$, и при конечных $k$ значение всегда строго меньше предела. Значит граница точная: для каждого $k$ конструкция выше даёт решётку, где жадный алгоритм набирает ровно $1 - \left(\frac{k-1}{k}\right)^k$ от оптимума, и лучше гарантировать нельзя.
