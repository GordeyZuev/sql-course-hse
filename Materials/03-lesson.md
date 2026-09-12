# **Lesson 3. JOIN и операции над множествами**

На прошлой паре мы считали показатели внутри одной таблицы. Но имя пассажира, цена перелёта и модель самолёта лежат в разных таблицах. Сегодня научимся соединять их.

---

## **Сегодня пройдемся по этому плану:**

1. **Alias таблицы** – короткое имя и запись `alias.column`.
2. **Ключи и условие** `ON`.
3. `INNER`, `LEFT`, `RIGHT`, `FULL OUTER JOIN`.
4. **Anti join, semi join и full exclusive join**.
5. `USING`; несколько условий в `ON`; отличие `ON` от `WHERE`.
6. `CROSS JOIN`, self join и non-equi join.
7. `UNION [ALL]`, `INTERSECT [ALL]`, `EXCEPT [ALL]`.

---

## **Часть 0. Для чего вообще нужны разные таблицы?**

Представим покупку авиабилета.

У нас есть пассажир, рейс, аэропорты, самолёт и цена. Можно записать всё в одну огромную таблицу, но тогда огромное количество данных (к примеру, имя аэропорта и модель самолёта) будет повторятся тысячи раз.



Поэтому данные разделены по смыслу:


| Таблица     | Что означает одна строка            |
| ----------- | ----------------------------------- |
| `airplanes` | один тип самолёта                   |
| `airports`  | один аэропорт                       |
| `flights`   | один рейс в конкретную дату         |
| `timetable` | один рейс вместе с данными маршрута |
| `tickets`   | один билет пассажира                |
| `segments`  | один перелёт из билета              |


**Зачем это нужно?** Представим, что модель самолёта записана прямо в каждом рейсе. Исправили название модели – пришлось менять тысячи строк. После нормализации код самолёта остаётся в расписании, а название хранится один раз в `airplanes`.

> **Нормализация** – разделение данных по сущностям, чтобы один факт хранился в одном месте.



Коротко о первых нормальных формах:


| Форма | Простая идея                                | Пример                                                            |
| ----- | ------------------------------------------- | ----------------------------------------------------------------- |
| 1НФ   | в одной ячейке – одно значение, а не список | сегменты билета лежат отдельными строками                         |
| 2НФ   | свойства строки зависят от всего её ключа   | цена в `segments` относится к паре «билет + рейс»                 |
| 3НФ   | описание другой сущности выносим отдельно   | модель самолёта хранится в `airplanes`, а маршрут содержит её код |


Формальные определения строже, но пока нам достаточно главной мысли: **каждый факт кладём туда, где он принадлежит своей сущности**. `JOIN` помогает собрать разделённые факты обратно для ответа.

В демобазе маршрут может изменяться со временем. Таблица `flights` хранит номер маршрута и время рейса, а аэропорты и самолёт относятся к `routes`.

Представление `timetable` уже соединило рейс с нужной версией маршрута. Поэтому из него можно сразу взять:

- `departure_airport`;
- `arrival_airport`;
- `airplane_code`.

---

## **Часть I. Псевдоним таблицы**

Во всех запросах, что мы писали, каждый столбец находится самостоятельно, нам не нужно писать из какой именно таблицы мы делаем запрос, так как именно с одной таблицей мы и имеем дело. Но когда мы начнем взаимодействовать с несколькими таблицами, начнутся проблемы! Поэтому давайте (до того, как мы дойдем до нашего первого `JOIN`) посмотрим как можно задавать полный путь аттрибута (столбца) таблицы!

Так мы выводили все столбцы.

```sql
SELECT *
FROM airplanes;
```

Теперь через алиас (или же псевдоним) назовём гораздо короче – `a`:

```sql
SELECT *
FROM airplanes AS a;
```

Результат тот же.  То есть мы договорились обозначать, что внутри запроса `a` будет обоначать `airplanes`.

> **Псевдоним таблицы (alias)** – короткое имя таблицы внутри одного запроса.

К колонке обращаемся через точку:

```sql
SELECT a.model
FROM airplanes AS a;
```

`a.model` читается как «колонка `model` из таблицы `a`».

### Все колонки одной таблицы

```sql
SELECT a.*
FROM airplanes AS a;
```

`a.*` означает «все колонки таблицы `a`».

Пока таблица одна, результат совпадает с `SELECT *`. После `JOIN` запись пригодится, чтобы взять все колонки только одной стороны.

В домашней работе перечисляем колонки явно, если условие не просит вывести все.

### Alias можно писать без `AS`

Эти запросы равнозначны:

```sql
SELECT a.model
FROM airplanes AS a;
```

```sql
SELECT a.model
FROM airplanes a;
```

Сначала будем писать `AS`: так момент появления нового имени виден лучше.

### Alias таблицы и alias колонки

```sql
SELECT a.model AS airplane_model
FROM airplanes AS a;
```

- `a` – короткое имя таблицы;
- `airplane_model` – имя колонки результата.

Псевдонимы живут только внутри одного запроса. Таблица и колонка в базе не переименовываются.

---

## **Часть II. JOIN `JOIN`-ом, но как объединять?**

Давайте посмотрим на таблицу `segments`. В ней есть номер билета, но нет имени пассажира:

```sql
SELECT ticket_no,
       flight_id,
       fare_conditions,
       price
FROM segments
ORDER BY ticket_no, flight_id
LIMIT 5;
```


Имя пассажира можно найти в таблице `tickets`. Заодно выведем `book_ref` – номер бронирования:

```sql
SELECT book_ref,
       ticket_no,
       passenger_name
FROM tickets
ORDER BY ticket_no
LIMIT 5;
```

В обеих таблицах есть `ticket_no` – номер одного билета:

```text
segments.ticket_no = tickets.ticket_no
```

> **Первичный ключ** – колонка или набор колонок, который однозначно определяет строку таблицы.
>
> **Внешний ключ** – колонка, которая ссылается на ключ другой таблицы.

`tickets.ticket_no` – первичный ключ, а `segments.ticket_no` ссылается на него.

### **Сначала выберем одно бронирование**

В одном бронировании может быть несколько билетов. Возьмём маленькое бронирование `0000EG`, чтобы результат можно было проверить глазами:

```sql
SELECT ticket_no,
       passenger_name
FROM tickets
WHERE book_ref = '0000EG'
ORDER BY ticket_no;
```

Получим два билета одного пассажира:

```text
0005434564147 | John Beland
0005434564152 | John Beland
```

Теперь соединим эти билеты с сегментами:

```sql
SELECT t.ticket_no,
       t.passenger_name,
       s.flight_id,
       s.price
FROM tickets AS t
JOIN segments AS s
  ON s.ticket_no = t.ticket_no
WHERE t.book_ref = '0000EG'
ORDER BY t.ticket_no, s.flight_id;
```

```text
0005434564147 | John Beland | 15113 | 12000.00
0005434564147 | John Beland | 15408 |  2500.00
0005434564152 | John Beland | 18015 | 17225.00
0005434564152 | John Beland | 18060 |  4225.00
```

`JOIN` нашёл строки с одинаковым `ticket_no` и поставил их колонки рядом.

Посмотрим на общую механику:

![Как работает JOIN](../Others/extra_pics/join_schema.png)

`JOIN` находит строки с подходящим ключом и ставит их колонки рядом.

Общая карта видов соединения:

![Типы соединений](../Others/extra_pics/join_types.png)

Синим показаны строки, которые попадут в результат.

На схеме смешаны две группы подписей:

- `INNER`, `LEFT`, `RIGHT`, `FULL` – ключевые слова PostgreSQL;
- `SEMI`, `ANTI`, `EXCLUSIVE` – названия приёмов, которые собираются из знакомых конструкций.

В Spark SQL и Hive встречаются явные `LEFT SEMI JOIN` / `LEFT ANTI JOIN` – **та же идея**, другой синтаксис. На экзамене по нашему курсу достаточно записи для PostgreSQL.

---



## **Часть III. Можно ли не писать имя таблицы?**

После `JOIN` имя колонки нужно указывать аккуратнее.

Да, если PostgreSQL понимает источник однозначно:

```sql
SELECT model
FROM airplanes AS a;
```

Колонка `model` есть только в `airplanes`, поэтому запрос выполнится.

В `segments` и `tickets` есть общий `ticket_no`. Такая запись неоднозначна:

```text
SELECT ticket_no
FROM segments AS s
JOIN tickets AS t
  ON t.ticket_no = s.ticket_no;
```

PostgreSQL не выбирает таблицу по смыслу. Он вернёт ошибку:

```text
column reference "ticket_no" is ambiguous
```

Нужно указать источник:

```sql
SELECT s.ticket_no,
       t.passenger_name
FROM segments AS s
JOIN tickets AS t
  ON t.ticket_no = s.ticket_no
WHERE t.book_ref = '0000EG';
```

Короткое правило:

- колонка встречается только в одном источнике – имя таблицы можно не писать;
- колонка есть в нескольких источниках – нужен `alias.column`;
- в учебных `JOIN` лучше указывать alias у всех колонок: запрос легче читать.

---



## **Часть IV.** `INNER JOIN`

Перед сравнением видов `JOIN` познакомимся с двумя маленькими таблицами. Они лежат в схеме `course` и специально устроены так, чтобы результат можно было проверить глазами.

Справочник аэропортов:

```sql
SELECT *
FROM course.airports_master
ORDER BY airport_code;
```

```text
DME | Domodedovo
SVO | Sheremetyevo
VKO | Vnukovo
```

Независимая выгрузка вылетов:

```sql
SELECT *
FROM course.departures_feed
ORDER BY departure_id;
```

```text
101 | SVO
102 | SVO
103 | VKO
104 | LED
```

`DME` есть только в справочнике, `LED` – только в выгрузке. `SVO` встречается в выгрузке дважды.

> `INNER JOIN` – оставляет только строки, для которых нашлась пара.

```sql
SELECT a.airport_code,
       a.airport_name,
       d.departure_id
FROM course.airports_master AS a
INNER JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
ORDER BY a.airport_code, d.departure_id;
```

```text
SVO | Sheremetyevo | 101
SVO | Sheremetyevo | 102
VKO | Vnukovo      | 103
```

`DME` и `LED` не образовали пару, поэтому исчезли. `SVO` появился дважды: одна строка слева нашла две строки справа.

```text
JOIN = INNER JOIN
```

Слово `INNER` можно опустить. Все следующие примеры будут менять только тип соединения – таблицы и условие останутся теми же.

### **Все колонки одной стороны**

Запись `a.*` означает «все колонки только таблицы `a`»:

```sql
SELECT a.*,
       d.departure_id
FROM course.airports_master AS a
JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
ORDER BY a.airport_code, d.departure_id;
```

---



## **Часть V.** `LEFT OUTER JOIN`

> `LEFT OUTER JOIN` – сохраняет все строки слева и добавляет найденные строки справа.

Поменяем в предыдущем запросе только тип соединения:

```sql
SELECT a.airport_code,
       a.airport_name,
       d.departure_id
FROM course.airports_master AS a
LEFT JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
ORDER BY a.airport_code, d.departure_id;
```

```text
DME | Domodedovo   | NULL
SVO | Sheremetyevo | 101
SVO | Sheremetyevo | 102
VKO | Vnukovo      | 103
```

`DME` не нашёл пары, но остался, потому что находится слева. Колонки правой таблицы для него получили `NULL`.

```text
LEFT JOIN = LEFT OUTER JOIN
```

### **Тот же `LEFT JOIN` на настоящих билетах**

Вернёмся к бронированию `0000EG`. Слева находятся его билеты, справа – сегменты:

```sql
SELECT t.ticket_no,
       t.passenger_name,
       s.flight_id
FROM tickets AS t
LEFT JOIN segments AS s
  ON s.ticket_no = t.ticket_no
WHERE t.book_ref = '0000EG'
ORDER BY t.ticket_no, s.flight_id;
```

Для этого бронирования каждому билету нашлись сегменты, поэтому `NULL` в результате нет. Но если бы сегмента не было, билет всё равно сохранился бы.

### **Несколько `LEFT JOIN` подряд**

Теперь добавим ещё одну таблицу. Соединённую таблицу можно снова соединить со следующей; предложения `JOIN` без скобок обрабатываются слева направо:

```sql
SELECT t.ticket_no,
       t.passenger_name,
       s.flight_id,
       f.status AS flight_status
FROM tickets AS t
LEFT JOIN segments AS s
  ON s.ticket_no = t.ticket_no
LEFT JOIN flights AS f
  ON f.flight_id = s.flight_id
WHERE t.book_ref = '0000EG'
ORDER BY t.ticket_no, s.flight_id
LIMIT 10;
```

Нет сегмента – колонки `s` будут `NULL`. Нет рейса – `NULL` в `f`. Билет слева всё равно останется.

---



## **Часть VI.** `RIGHT OUTER JOIN`

> `RIGHT OUTER JOIN` – сохраняет все строки справа и добавляет найденные строки слева.

Переставим таблицы из `LEFT JOIN`: справочник теперь справа.

```sql
SELECT a.airport_code,
       a.airport_name,
       d.airport_code AS feed_airport,
       d.departure_id
FROM course.departures_feed AS d
RIGHT JOIN course.airports_master AS a
  ON d.airport_code = a.airport_code
ORDER BY a.airport_code, d.departure_id;
```

```text
DME | Domodedovo   | NULL | NULL
SVO | Sheremetyevo | SVO  | 101
SVO | Sheremetyevo | SVO  | 102
VKO | Vnukovo      | VKO  | 103
```

Получили тот же смысл, что в предыдущем `LEFT JOIN`:

```text
first LEFT JOIN second
=
second RIGHT JOIN first
```

```text
RIGHT JOIN = RIGHT OUTER JOIN
```

На практике `RIGHT JOIN` встречается реже. Обычно таблицы меняют местами и пишут цепочку через `LEFT JOIN`: так запрос читается слева направо.

---



## **Часть VII.** `FULL OUTER JOIN`

> `FULL OUTER JOIN` – сохраняет все строки обеих сторон.

Как `LEFT` и `RIGHT`, но теперь не выбрасываем ни одну сторону:

```sql
SELECT a.airport_code AS master_airport,
       a.airport_name,
       d.airport_code AS feed_airport,
       d.departure_id
FROM course.airports_master AS a
FULL JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
ORDER BY master_airport,
         feed_airport,
         d.departure_id;
```

```text
DME  | Domodedovo   | NULL | NULL
SVO  | Sheremetyevo | SVO  | 101
SVO  | Sheremetyevo | SVO  | 102
VKO  | Vnukovo      | VKO  | 103
NULL | NULL          | LED  | 104
```

В результате видны все три ситуации:

- `DME` есть только слева;
- `SVO` и `VKO` нашли пары;
- `LED` есть только справа.

```text
FULL JOIN = FULL OUTER JOIN
```

> `FULL JOIN` не равен `UNION ALL`: join ставит колонки рядом и ищет пары по `ON`. `UNION ALL` позже поставит строки друг под другом.

---



## **Часть VIII. Anti join**

> **Anti join** отвечает на вопрос: «Каким строкам не нашлась ни одна пара?»

### **Left anti join**

Сохраним справочник через `LEFT JOIN`, а затем оставим строку с `NULL` справа:

```sql
SELECT a.airport_code,
       a.airport_name
FROM course.airports_master AS a
LEFT JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
WHERE d.airport_code IS NULL;
```

```text
DME | Domodedovo
```

`DME` сохранился слева, но пары справа не получил.

### **Right anti join**

Теперь сохраним выгрузку справа и найдём код, которого нет в справочнике:

```sql
SELECT d.airport_code,
       d.departure_id
FROM course.airports_master AS a
RIGHT JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
WHERE a.airport_code IS NULL;
```

```text
LED | 104
```

Отдельных операторов `LEFT ANTI JOIN` и `RIGHT ANTI JOIN` в PostgreSQL нет. Это названия способов рассуждать.

---



## **Часть IX. Full exclusive join**

> **Full exclusive join** – строки, которые есть только на одной из двух сторон.

Возьмём `FULL JOIN` из части VII и добавим один фильтр:

```sql
SELECT a.airport_code AS master_airport,
       d.airport_code AS feed_airport
FROM course.airports_master AS a
FULL JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
WHERE a.airport_code IS NULL
   OR d.airport_code IS NULL
ORDER BY master_airport,
         feed_airport;
```

```text
DME  | NULL
NULL | LED
```

Совпадения `SVO` и `VKO` исчезли. Остались значения только слева и только справа. `FULL EXCLUSIVE JOIN` – название приёма, а не отдельная команда PostgreSQL.

---



## **Часть X. Semi join**

> **Semi join** отвечает на вопрос: «У каких строк есть хотя бы одна пара?»

Сначала попробуем знакомый `INNER JOIN`, но выведем только колонки справочника:

```sql
SELECT DISTINCT a.airport_code,
       a.airport_name
FROM course.airports_master AS a
JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
ORDER BY a.airport_code;
```

```text
SVO | Sheremetyevo
VKO | Vnukovo
```

Без `DISTINCT` обычный JOIN вернул бы `SVO` дважды. Для проверки существования пары есть более точная запись – `EXISTS`:

```sql
SELECT a.airport_code,
       a.airport_name
FROM course.airports_master AS a
WHERE EXISTS (
    SELECT 1
    FROM course.departures_feed AS d
    WHERE d.airport_code = a.airport_code
)
ORDER BY a.airport_code;
```

Результат тот же: `SVO` и `VKO`.

Для каждой строки `airports_master` PostgreSQL спрашивает: вернул ли внутренний запрос хотя бы одну строку?

- `SELECT 1` – заглушка: наружу единица не выводится;
- `d.airport_code = a.airport_code` связывает внутренний запрос с текущим аэропортом;
- для `EXISTS` достаточно найти одну строку.

Semi и anti – зеркальные вопросы:

| Semi | Anti |
| --- | --- |
| есть хотя бы одна пара | нет ни одной пары |
| `EXISTS (...)` | `NOT EXISTS (...)` или `LEFT JOIN` + `IS NULL` |


---



## **Часть XI.** `USING`, несколько условий в `ON` и `WHERE`

### **Короткая запись `USING`**

Если колонка связи называется одинаково в обеих таблицах, равенство можно сократить:

```sql
SELECT airport_code,
       airport_name,
       departure_id
FROM course.airports_master
JOIN course.departures_feed USING (airport_code)
ORDER BY airport_code, departure_id;
```

```text
SVO | Sheremetyevo | 101
SVO | Sheremetyevo | 102
VKO | Vnukovo      | 103
```

```text
USING (airport_code)
```

означает равенство одноимённых колонок. В результате общий `airport_code` выводится один раз.

`NATURAL JOIN` идёт ещё дальше и автоматически связывает все одноимённые колонки. Это рискованно: новая колонка в схеме может незаметно изменить результат. Поэтому в курсе используем явные `ON` и `USING`.

### **Несколько условий в `ON`**

> `ON` – логическое выражение. Две строки образуют пару, если всё выражение возвращает `TRUE`.

Условия можно соединять через `AND` и `OR`. Оставим только события с номерами до 102:

```sql
SELECT a.airport_code,
       a.airport_name,
       d.departure_id
FROM course.airports_master AS a
LEFT JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
 AND d.departure_id <= 102
ORDER BY a.airport_code, d.departure_id;
```

```text
DME | Domodedovo   | NULL
SVO | Sheremetyevo | 101
SVO | Sheremetyevo | 102
VKO | Vnukovo      | NULL
```

`VKO` есть в выгрузке, но его событие `103` не прошло второе условие `ON`. Поэтому справа появились `NULL`.

### **Почему не перенести условие в `WHERE`?**

```sql
SELECT a.airport_code,
       a.airport_name,
       d.departure_id
FROM course.airports_master AS a
LEFT JOIN course.departures_feed AS d
  ON d.airport_code = a.airport_code
WHERE d.departure_id <= 102
ORDER BY a.airport_code, d.departure_id;
```

```text
SVO | Sheremetyevo | 101
SVO | Sheremetyevo | 102
```

Условие в `ON` проверяется во время поиска пары. Условие в `WHERE` фильтрует уже готовый результат. Проверка `NULL <= 102` не даёт `TRUE`, поэтому строки `DME` и `VKO` исчезли.

---



## **Часть XII.** `CROSS JOIN`

> `CROSS JOIN` – соединение каждой строки слева с каждой строкой справа.

Возьмём три аэропорта и два класса обслуживания:

```sql
SELECT a.airport_code,
       f.fare_conditions
FROM course.airports_master AS a
CROSS JOIN course.fare_classes AS f
ORDER BY a.airport_code, f.fare_conditions;
```

```text
DME | Business
DME | Economy
SVO | Business
SVO | Economy
VKO | Business
VKO | Economy
```

Получили `3 × 2 = 6` строк. У `CROSS JOIN` нет условия `ON`: нужны все комбинации.

Если слева 1000 строк и справа 1000 строк, получится миллион. Поэтому используем его только тогда, когда все комбинации действительно нужны.

---

## **Часть XIII. Self join**

Иногда нужно сравнить строки одной таблицы между собой. В `departures_feed` у `SVO` есть два события. Найдём их пару:

```sql
SELECT d1.airport_code,
       d1.departure_id AS first_departure,
       d2.departure_id AS second_departure
FROM course.departures_feed AS d1
JOIN course.departures_feed AS d2
  ON d1.airport_code = d2.airport_code
 AND d1.departure_id < d2.departure_id;
```

```text
SVO | 101 | 102
```

> **Self join** – соединение таблицы с самой собой. Два alias позволяют обратиться к двум разным строкам одной таблицы.

- `d1.airport_code = d2.airport_code` – аэропорт совпадает;
- `d1.departure_id < d2.departure_id` – событие не соединяется само с собой, а пара не повторяется наоборот.

Если оставить только равенство аэропортов, появятся `(101, 101)`, `(102, 102)`, `(101, 102)` и `(102, 101)`. Условие `<` оставляет одну нужную пару.

---


## **Часть XIV. Non-equi / theta join**

До сих пор строки соединялись по равенству:

```text
left.key = right.key
```

Но условие `ON` может содержать другое сравнение.

**Вопрос: «Какие модели летают дальше, чем Bombardier CRJ700?»**

```sql
SELECT shorter.model AS base_model,
       shorter.range AS base_range,
       longer.model AS longer_model,
       longer.range AS longer_range
FROM airplanes AS shorter
JOIN airplanes AS longer
  ON longer.range > shorter.range
WHERE shorter.airplane_code = 'CR7'
ORDER BY longer.range, longer.airplane_code;
```

> **Equi join** (от английского `equality` – равенство) – соединение по `=`. Это самый частый случай.

> **Non-equi join** – в `ON` главное условие не равенство, а `<`, `>`, `<=`, `>=` или другой предикат.

> **Theta join** – общее имя соединения по произвольному условию в `ON`. Equi join – его частный случай.

Здесь одновременно используются self join и условие `>`.

---



## **Часть XV.** `UNION [ALL]`

`JOIN` ставит колонки рядом и ищет пары по `ON`. Операции над множествами ставят результаты нескольких `SELECT` друг под другом.

> `UNION [ALL]` – объединяет строки нескольких `SELECT`. Без `ALL` повторы удаляются, с `ALL` – сохраняются.

Сначала вариант без `ALL`:

```sql
SELECT airport_code
FROM course.airports_master
UNION
SELECT airport_code
FROM course.departures_feed
ORDER BY airport_code;
```

```text
DME
LED
SVO
VKO
```

`SVO` и `VKO` есть в обеих таблицах, но `UNION` оставил каждый код один раз.

### **`UNION ALL`**

Теперь добавим `ALL`, не меняя входные данные:

```sql
SELECT airport_code
FROM course.airports_master
UNION ALL
SELECT airport_code
FROM course.departures_feed
ORDER BY airport_code;
```

```text
DME
LED
SVO
SVO
SVO
VKO
VKO
```

Все строки сохранились: один `SVO` пришёл из справочника, ещё два – из выгрузки.

`UNION` тратит время на удаление дубликатов, поэтому на больших данных обычно медленнее `UNION ALL`.

У частей должны совпадать число, порядок и совместимые типы колонок. Имена результата берутся из первого `SELECT`. Общий `ORDER BY` ставится после последней части.

### **Реальный пример `UNION ALL`**

Соберём поток плановых и фактических вылетов:

```sql
SELECT flight_id,
       scheduled_departure AS event_time,
       'scheduled_departure' AS event_type
FROM flights
WHERE status = 'Arrived'
  AND flight_id <= 10

UNION ALL

SELECT flight_id,
       actual_departure AS event_time,
       'actual_departure' AS event_type
FROM flights
WHERE status = 'Arrived'
  AND actual_departure IS NOT NULL
  AND flight_id <= 10

ORDER BY flight_id, event_type;
```

---

## **Часть XVI.** `INTERSECT [ALL]`

> `INTERSECT [ALL]` – оставляет строки, которые присутствуют в обоих результатах. С `ALL` учитывается число повторов.

```sql
SELECT airport_code
FROM course.departures_feed
WHERE departure_id IN (101, 102, 103)
INTERSECT
SELECT airport_code
FROM course.departures_feed
WHERE departure_id IN (101, 102, 104)
ORDER BY airport_code;
```

```text
SVO
```

Обе части содержат `SVO`, поэтому он попал в результат. Хотя `SVO` встречается дважды с обеих сторон, обычный `INTERSECT` удалил повтор.

### **`INTERSECT ALL`**

Теперь добавим `ALL` к тому же запросу:

```sql
SELECT airport_code
FROM course.departures_feed
WHERE departure_id IN (101, 102, 103)
INTERSECT ALL
SELECT airport_code
FROM course.departures_feed
WHERE departure_id IN (101, 102, 104)
ORDER BY airport_code;
```

```text
SVO
SVO
```

`SVO` встречается дважды в обеих частях, поэтому `INTERSECT ALL` сохранил две копии. Обычный `INTERSECT` оставил бы одну.

### **Реальный пример**

Какие коды среди первых десяти рейсов встречались и как отправление, и как прибытие?

```sql
SELECT departure_airport AS airport_code
  FROM timetable
WHERE flight_id <= 10

INTERSECT

SELECT arrival_airport AS airport_code
  FROM timetable
WHERE flight_id <= 10
ORDER BY airport_code;
```

---

## **Часть XVII.** `EXCEPT [ALL]`

> `EXCEPT [ALL]` – оставляет строки первого результата, которых нет во втором. С `ALL` учитывается число повторов.

```sql
SELECT airport_code
FROM course.departures_feed
EXCEPT
SELECT airport_code
FROM course.airports_master
ORDER BY airport_code;
```

```text
LED
```

`LED` есть в выгрузке, но отсутствует в справочнике. Обычный `EXCEPT` удаляет повторы.

### **`EXCEPT ALL`**

Добавим `ALL` к тому же запросу:

```sql
SELECT airport_code
FROM course.departures_feed
EXCEPT ALL
SELECT airport_code
FROM course.airports_master
ORDER BY airport_code;
```

```text
LED
SVO
```

`SVO` встретился слева дважды, а справа один раз. Одна копия «вычлась», вторая осталась.

Направление важно: `A EXCEPT [ALL] B` не равно `B EXCEPT [ALL] A`.

### **Реальный пример**

Какие коды среди первых десяти рейсов встречались как отправление, но не встречались как прибытие?

```sql
SELECT departure_airport AS airport_code
FROM timetable
WHERE flight_id <= 10
EXCEPT
SELECT arrival_airport AS airport_code
FROM timetable
WHERE flight_id <= 10
ORDER BY airport_code;
```

---



## **Шпаргалочка**

| Название              | Запись в PostgreSQL                       | Что остаётся                            |
| --------------------- | ----------------------------------------- | --------------------------------------- |
| inner join            | `INNER JOIN` или `JOIN`                   | только найденные пары                   |
| left outer join       | `LEFT JOIN`                               | все слева и совпадения справа           |
| right outer join      | `RIGHT JOIN`                              | все справа и совпадения слева           |
| full outer join       | `FULL JOIN`                               | все строки обеих сторон                 |
| left anti join        | `LEFT JOIN` + `WHERE right.key IS NULL`   | строки слева без пары                   |
| right anti join       | `RIGHT JOIN` + `WHERE left.key IS NULL`   | строки справа без пары                  |
| full exclusive join   | `FULL JOIN` + проверка `NULL`             | все несовпавшие строки                  |
| left semi join        | `WHERE EXISTS (...)`                      | строки слева, у которых есть пара       |
| cross join            | `CROSS JOIN`                              | все комбинации                          |
| self join             | одна таблица с двумя alias                | строки таблицы сравниваются между собой |
| non-equi / theta join | `JOIN ... ON` с `<`, `>`, другим условием | пары по произвольному условию           |

| Операция | Что делает |
| --- | --- |
| `UNION [ALL]` | складывает строки; с `ALL` сохраняет повторы |
| `INTERSECT [ALL]` | оставляет общие строки |
| `EXCEPT [ALL]` | оставляет строки первой части, которых нет во второй |

---



## **Итог**

Сегодня мы:
- соединяли таблицы (очень старались);
- дали таблице короткое имя;
- выяснили, когда колонку можно писать без alias;
- вывели все колонки одной стороны через `alias.*`;
- связали первичный и внешний ключ через `ON`;
- разобрали `INNER`, `LEFT`, `RIGHT` и `FULL OUTER JOIN`;
- получили строки без пары через anti join;
- получили строки с парой через semi join;
- оставили все несовпадения через full exclusive join;
- построили все комбинации через `CROSS JOIN`;
- сравнили строки одной таблицы через self join;
- соединили строки по неравенству через non-equi join;
- поставили строки друг под другом через `UNION [ALL]`;
- нашли пересечение через `INTERSECT [ALL]`;
- нашли разность через `EXCEPT [ALL]`.

На следующей паре будем сравнивать строку с соседними строками внутри группы – появятся оконные функции.

## **Полезное**

- [Соединения таблиц](https://postgrespro.ru/docs/postgresql/current/queries-table-expressions#QUERIES-JOIN)
- [Псевдонимы таблиц и колонок](https://postgrespro.ru/docs/postgresql/current/queries-table-expressions#QUERIES-TABLE-ALIASES)
- [Подзапросы и EXISTS](https://postgrespro.ru/docs/postgresql/current/functions-subquery)
- [UNION, INTERSECT и EXCEPT](https://postgrespro.ru/docs/postgresql/current/queries-union)
- [Демобаза «Авиаперевозки»](https://postgrespro.ru/docs/postgrespro/current/demodb-bookings)
- [Маленькие учебные таблицы](../Others/course_tables.md)

