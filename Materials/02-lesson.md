# **Lesson 2. Агрегаты, GROUP BY и подзапросы**

На первой паре мы работали с отдельными строками. Сегодня научимся сжимать таблицу в **показатели** – сколько, в среднем, какой край расписания, кто выше порога.

---

## **Сегодня пройдемся по этому плану:**

1. **Зачем** сжимать данные?
2. **Агрегатные функции** – `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.
3. `COUNT(*)` **и** `COUNT(column)` и связь с NULL.
4. `GROUP BY`.
5. **Расширенный конвейер** – куда писать `GROUP BY` и `HAVING`.
6. `WHERE` **vs** `HAVING` – фильтр строк и фильтр групп.
7. `**MIN` и `MAX**` – самый ранний и самый поздний в наборе.
8. **Scalar-подзапрос** – один порог из всей таблицы.
9. `COUNT(DISTINCT ...)` – уникальные значения внутри группы.
10. `FILTER` – условный агрегат без лишнего `CASE`.
11. **Задержка в минутах** – `extract` и среднее по группе.
12. `string_agg` – склеить значения группы в одну строку.
13. **Подзапрос во** `FROM` – промежуточная таблица внутри запроса.
14. `WITH` **(CTE)** – именованный шаг перед финальным `SELECT`.

---

## **Часть 0. От строки к показателю**

### Про определение строки

На паре 1 мы ввели главный вопрос курса:

```text
Что означает одна строка
```

Сегодня мы поработаем с другим смыслом. Исходная таблица `flights` – это «одна строка = один рейс». После `GROUP BY status` – «одна строка = один статус и его показатели».

<p>
<img src="../Others/extra_pics/cats.png" width="880" alt="Строки таблицы до сжатия" style="display: block; margin: 0.75em 0;" />
<img src="../Others/extra_pics/grouped_cats.png" width="880" alt="После GROUP BY — отдельный мешок на группу" style="display: block; margin: 0.75em 0;" />
</p>

Кратко – таблицы, с которыми будем работать чаще всего:


| Таблица     | Grain одной строки             | Пример вопроса сегодня            |
| ----------- | ------------------------------ | --------------------------------- |
| `flights`   | один рейс                      | сколько рейсов в каждом `status`? |
| `timetable` | один рейс (расписание + факты) | сколько вылетов из аэропорта?     |
| `segments`  | один сегмент билета на рейсе   | средняя цена по классу тарифа?    |
| `tickets`   | один билет                     | сколько билетов в бронировании?   |
| `airports`  | один аэропорт                  | сколько аэропортов в стране?      |


### Зачем сжимать в SQL, а не в Python

Представьте вопрос руководителя: «Сколько рейсов сейчас в каждом статусе?»

На паре 1 мы могли бы выгрузить все строки `flights` и посчитать где-то в ином месте (к примеру в Pandas / python). SQL умеет сжать таблицу **внутри запроса**:

```sql
SELECT status, count(*) AS flight_count
FROM flights
GROUP BY status
ORDER BY flight_count DESC, status;
```

Разберем по шагам – сначала агрегаты без группировки, потом `GROUP BY`, потом фильтры групп.

На первой паре конвейер был:

```text
FROM → WHERE → SELECT → DISTINCT → ORDER BY → LIMIT → OFFSET
```

Сегодня между `WHERE` и `SELECT` появляются **группировка** и **фильтр групп**:

```text
FROM → WHERE → (NEW) GROUP BY → (NEW) HAVING → SELECT → DISTINCT → ORDER BY → LIMIT → OFFSET
```

Теперь сначала мы отбираем строки, потом сжимаем, потом отбрасываем группы, потом формируем список колонок результата.

---

## **Часть I. Агрегатные функции**

> **Агрегатная функция** – вычисление над **набором строк**, которое возвращает **одно** значение: число, сумму, среднее, минимум или максимум...

Наиболее используемые варианты:


| Функция      | Вопрос, на который отвечает |
| ------------ | --------------------------- |
| `COUNT(...)` | сколько                     |
| `SUM(...)`   | какая сумма                 |
| `AVG(...)`   | какое среднее               |
| `MIN(...)`   | какое наименьшее            |
| `MAX(...)`   | какое наибольшее            |


Попробуем на пальцах. Есть пять цен сегментов: 1000, 2000, 3000, 4000, 5000.

- `COUNT(*)` → 5 (пять строк);
- `SUM(price)` → 15000;
- `AVG(price)` → 3000;
- `MIN(price)` → 1000;
- `MAX(price)` → 5000.

SQL делает то же самое, только набор строк берёт из таблицы.

### Один агрегат без `GROUP BY`

Если в `SELECT` только агрегаты и **нет** `GROUP BY`, вся таблица (после `WHERE`) сжимается в **одну строку**:

```sql
SELECT count(*) AS total_flights,
       min(scheduled_departure) AS earliest_departure,
       max(scheduled_departure) AS latest_departure
FROM flights;
```

Одна строка – один «мешок» из всех подходящих рейсов.

### Несколько агрегатов сразу

Их можно комбинировать в одном `SELECT`:

```sql
SELECT count(*) AS segment_count,
       round(avg(price), 2) AS avg_price,
       min(price) AS min_price,
       max(price) AS max_price
FROM segments;
```

`round(..., 2)` – округление **итогового** значения до двух знаков. Округлять каждую цену до `AVG` – другая задача и другой результат.

### `SUM` и `AVG` на деньгах

В `bookings` цены – `numeric`. Для сумм и средних это удобно: нет накопления ошибки, как у `real`.

Пока **без** `GROUP BY` – одна строка по всей таблице `segments`:

```sql
SELECT count(*) AS segment_count,
       sum(price) AS total_revenue,
       round(avg(price), 2) AS avg_segment_price
FROM segments;
```

Grain: вся таблица сегментов одним «мешком». Когда понадобится разрез по классу тарифа – добавим `GROUP BY` в части IV.

---

## **Часть II.** `COUNT(*)` **и** `COUNT(column)`

`COUNT` – самый частый агрегат и самый коварный.

### `COUNT(*)` – число строк

Считает **строки**, не смотря на NULL в отдельных колонках:

```sql
SELECT count(*) AS all_rows
FROM flights;
```

### `COUNT(column)` – число non-NULL

Игнорирует строки, где выражение `NULL`:

```sql
SELECT count(*) AS all_rows,
       count(actual_arrival) AS known_arrivals,
       count(*) - count(actual_arrival) AS unknown_arrivals
FROM flights;
```

Три числа в одной строке – срез по полноте данных. Для ML это напоминание: «неизвестно» – не ноль.

### Когда что выбирать


| Задача                              | Что писать      |
| ----------------------------------- | --------------- |
| «сколько строк в таблице / группе»  | `COUNT(*)`      |
| «сколько строк, где поле заполнено» | `COUNT(column)` |


---

## **Часть III. NULL в агрегатах**

Правило для `SUM`, `AVG`, `MIN`, `MAX`: **NULL не участвует** в расчёте (!)

```sql
SELECT avg(actual_arrival - scheduled_arrival) AS avg_delay
FROM flights
WHERE status = 'Arrived';
```

Рейсы без `actual_arrival` в среднее не попадут – как если бы их не было в наборе.

Если non-NULL значений нет, `AVG` вернёт `NULL`, а не ноль. Ноль – это «средняя задержка ровно ноль», совсем другой смысл.

- `COUNT(*)` от пустой таблицы даст `0`.
- `COUNT(column)` на пустом наборе или когда все NULL – тоже `0`.

---

## **Часть IV.** `GROUP BY`

> `GROUP BY` – правило, по каким колонкам (или выражениям) разбить строки на **группы**. В каждой группе агрегаты считаются отдельно.

```sql
SELECT status, count(*) AS flight_count
FROM flights
GROUP BY status
ORDER BY flight_count DESC, status;
```

### Что можно писать в `SELECT`

Либо колонка входит в `GROUP BY`, либо она внутри агрегата. Иначе PostgreSQL не понимает, какое значение показать из группы:

```sql
-- Ошибка: departure_airport не в GROUP BY и не в агрегате.
SELECT departure_airport, status, count(*)
FROM flights
GROUP BY status;
```

Исключение – функциональная зависимость в PostgreSQL: если сгруппировали по первичному ключу, остальные колонки той же таблицы иногда допустимы.

### Несколько колонок в группировке

```sql
-- grain: пара аэропортов.
SELECT departure_airport,
       arrival_airport,
       count(*) AS flight_count
FROM timetable
GROUP BY departure_airport, arrival_airport
ORDER BY flight_count DESC, departure_airport, arrival_airport
LIMIT 10;
```

### Группировка по выражению

Можно сжимать не только по колонке, но и по вычислению:

```sql
-- grain: календарный день вылета.
SELECT scheduled_departure::date AS departure_date,
       count(*) AS flight_count
FROM flights
GROUP BY scheduled_departure::date
ORDER BY flight_count DESC, departure_date
LIMIT 10;
```

Выражение в `GROUP BY` и псевдоним в `SELECT` – **одно и то же** по смыслу. В `ORDER BY` можно ссылаться на alias `departure_date`.

### Сортировка по агрегату

```sql
SELECT country, count(*) AS airport_count
FROM airports
GROUP BY country
ORDER BY airport_count DESC, country
LIMIT 15;
```

Как и на паре 1: при равных значениях добавляйте tie-breaker (`country`), иначе порядок не зафиксирован контрактом.

### Несколько агрегатов в одной группе

Теперь вернёмся к ценам сегментов – но уже **по классу тарифа**:

```sql
SELECT fare_conditions,
       count(*) AS segment_count,
       round(avg(price), 2) AS avg_price,
       max(price) AS max_price
FROM segments
GROUP BY fare_conditions
ORDER BY fare_conditions;
```

Одна строка – не один сегмент, а **статистика по всем сегментам класса**. Имена колонок `segment_count`, `avg_price` – часть контракта в HW2: подписывайте так, как просит условие.

### `SUM` по группе: цена билета целиком

Частый вопрос: «сколько пассажир заплатил за билет в сумме по всем сегментам?»

```sql
SELECT ticket_no,
       count(*) AS segment_count,
       sum(price) AS total_price
FROM segments
WHERE flight_id <= 5000
GROUP BY ticket_no
ORDER BY total_price DESC, ticket_no
LIMIT 10;
```

Здесь `SUM` складывает цены **внутри группы** `ticket_no`. `COUNT(*)` в той же группе – сколько сегментов у билета. Дальше по таким группам часто сортируют по сумме или отбирают порогом в `HAVING`.

### Билеты в одном бронировании

Та же логика `count(*)`, другой grain: в `tickets` одна строка – один билет, в результате одна строка – одно бронирование и сколько в нём билетов:

```sql
SELECT book_ref,
       count(*) AS ticket_count
FROM tickets
WHERE book_ref < '100000'
GROUP BY book_ref
ORDER BY ticket_count DESC, book_ref
LIMIT 10;
```

---

## **Часть V. Расширенный конвейер**

Полная логическая модель одного `SELECT` без `JOIN`:

```text
FROM
  → WHERE        -- отбор исходных строк
  → GROUP BY     -- сбор групп
  → HAVING       -- отбор групп
  → SELECT       -- список колонок и агрегатов
  → DISTINCT     -- убрать дубли результата (редко с агрегатами)
  → ORDER BY
  → LIMIT / OFFSET
```

Перед запуском полезно проговорить вслух: «сначала берём таблицу, потом режем строки, потом группируем…».

`HAVING` выполняется **после** группировки. В нём уже можно ссылаться на агрегаты:

```sql
SELECT country, count(*) AS airport_count
FROM airports
GROUP BY country
HAVING count(*) >= 10
ORDER BY airport_count DESC, country;
```

---

## **Часть VI.** `WHERE` **vs** `HAVING`


|               | `WHERE`                | `HAVING`               |
| ------------- | ---------------------- | ---------------------- |
| Когда         | до группировки         | после группировки      |
| Что фильтрует | исходные строки        | уже построенные группы |
| Агрегаты      | нельзя (ещё нет групп) | можно                  |


Пример: среди **прибывших** рейсов найти аэропорты с большим числом вылетов.

```sql
SELECT departure_airport, count(*) AS arrived_count
FROM timetable
WHERE status = 'Arrived'
GROUP BY departure_airport
HAVING count(*) >= 100
ORDER BY arrived_count DESC, departure_airport;
```

- `WHERE status = 'Arrived'` – выкидываем неприбывшие рейсы **до** счёта.
- `HAVING count(*) >= 100` – оставляем только аэропорты, где набралось достаточно строк **в группе**.

Типичная ошибка – написать `HAVING status = 'Arrived'`. Формально иногда сработает, но мыслить так неудобно: статус – свойство строки, его место в `WHERE`.

Другая ошибка – `WHERE count(*) >= 100`. Агрегат в `WHERE` запрещён: групп ещё нет.

### Один фильтр – два места?

Иногда условие можно перенести. `WHERE price > 1000` до `GROUP BY` и `HAVING sum(price) > 1000` после – **разные** вопросы. Первое режет строки, второе – итог по группе.

### Будущие рейсы по аэропорту

Вопрос: сколько **ещё не вылетевших** рейсов со статусом `Scheduled` запланировано из каждого аэропорта? Прошлое отсекаем в `WHERE` **до** группировки; «сейчас» в демобазе – `bookings.now()`, не системный `now()`:

```sql
SELECT departure_airport,
       count(*) AS scheduled_count
FROM timetable
WHERE status = 'Scheduled'
  AND scheduled_departure > bookings.now()
GROUP BY departure_airport
ORDER BY scheduled_count DESC, departure_airport
LIMIT 10;
```

Grain: один аэропорт отправления и число подходящих рейсов в его группе.

---

## **Часть VII.** `MIN` **и** `MAX` **— края набора**

`MIN` и `MAX` отвечают на простой вопрос: **какое значение в наборе самое маленькое и самое большое?** Для дат это часто «самый ранний» и «самый поздний» момент.

Без `GROUP BY` края считаются по **всей** таблице (после `WHERE`, если он есть) — снова одна строка результата:

```sql
-- grain: вся таблица flights.
SELECT min(scheduled_departure) AS earliest_departure,
       max(scheduled_departure) AS latest_departure
FROM flights;
```

Сузим срез — и `MIN`/`MAX` пересчитаются только по отобранным рейсам:

```sql
SELECT min(scheduled_departure) AS earliest_departure,
       max(scheduled_departure) AS latest_departure
FROM flights
WHERE status = 'Scheduled';
```

### Внутри группы

К `count(*)` в `GROUP BY` можно добавить «хвост» расписания внутри категории:

```sql
-- grain: один status.
SELECT status,
       count(*) AS flight_count,
       max(scheduled_departure) AS latest_scheduled_departure
FROM flights
GROUP BY status
ORDER BY flight_count DESC, status;
```

`MIN` и `MAX`, как `AVG`, **не учитывают** `NULL`: если у части рейсов время неизвестно, края считаются только по заполненным строкам.

> В HW2 среднюю цену иногда просят как `round(avg(price), 2)` — округляйте **уже среднее**, не каждую цену в таблице. Отдельной «лекции про проценты» на этой паре нет: доли и `100.0` разберём позже, когда понадобятся в отчётах.

---



## **Часть VIII. Scalar-подзапрос**

> **Scalar-подзапрос** – подзапрос в скобках, который возвращает **ровно одно значение** (одна строка, одна колонка). Его можно подставить туда, где ждут одно число или строку.



Классический пример – порог «выше среднего»:

```sql
SELECT ticket_no, flight_id, price
FROM segments
WHERE price > (SELECT avg(price) FROM segments)
ORDER BY price DESC, ticket_no, flight_id
LIMIT 20;
```

Внутренний `SELECT avg(price) FROM segments` не видит внешний `WHERE` – это **общее** среднее по всей таблице `segments`.



Тот же приём в `HAVING`:

```sql
SELECT fare_conditions, round(avg(price), 2) AS avg_price
FROM segments
GROUP BY fare_conditions
HAVING avg(price) > (SELECT avg(price) FROM segments)
ORDER BY avg_price DESC, fare_conditions;
```



### Что если подзапрос вернёт две строки?

PostgreSQL выдаст ошибку: scalar ожидает одно значение. Это защита от неоднозначности.



### Подзапрос в `SELECT`

Scalar можно вывести **рядом** с групповым агрегатом — как ориентир «максимум по всей таблице»:

```sql
SELECT fare_conditions,
       round(avg(price), 2) AS avg_price,
       (SELECT max(price) FROM segments) AS max_price_in_dataset
FROM segments
GROUP BY fare_conditions
ORDER BY fare_conditions;
```

Внутренний запрос не зависит от текущей группы: это один общий `max` для всего `segments`.

---

## **Часть IX.** `COUNT(DISTINCT ...)`

Иногда в группе нужно число **разных** значений, а не строк:

```sql
SELECT flight_id, count(DISTINCT ticket_no) AS ticket_count
FROM segments
WHERE flight_id <= 1000
GROUP BY flight_id
ORDER BY ticket_count DESC, flight_id
LIMIT 10;
```

`COUNT(DISTINCT column)` – сколько **разных** значений встретилось в группе. На рейсе несколько сегментов могут относиться к одному билету: `count(*)` посчитает строки, `count(DISTINCT ticket_no)` – пассажирские билеты.

Не путать с `DISTINCT` в начале `SELECT`: тот убирает дубли **готовых строк результата**, а `COUNT(DISTINCT ...)` – считает уникальные значения **внутри агрегата**.

---

## **Часть X.** `FILTER` **– условный агрегат**

До PostgreSQL 9.4 часто писали `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`. Синтаксис `FILTER` короче и читается как «посчитай, но только по подмножеству»:

```sql
SELECT status,
       count(*) AS flight_count,
       count(*) FILTER (WHERE actual_departure IS NOT NULL) AS departed_count
FROM flights
GROUP BY status
ORDER BY status;
```

В одном `SELECT` – несколько агрегатов с разными условиями, без повторного чтения таблицы.

Для сумм и средних:

```sql
SELECT departure_airport,
       count(*) AS arrived_count,
       count(*) FILTER (WHERE actual_arrival > scheduled_arrival) AS delayed_count
FROM timetable
WHERE status = 'Arrived'
GROUP BY departure_airport
HAVING count(*) >= 50
ORDER BY delayed_count DESC, departure_airport
LIMIT 10;
```

`FILTER` – часть **агрегата**, не замена `WHERE`. `WHERE` режет входные строки до группировки; `FILTER` выбирает, какие строки группы попадают в конкретный счётчик.

---

## **Часть XI. Задержка в минутах**

На паре 1 мы вычитали моменты времени. Разность двух `timestamptz` – это `interval` (например «2 hours 15 minutes»). Для отчёта часто нужно **одно число** – минуты.

**Шаг 1.** Задержка по каждому рейсу – интервал:

```sql
SELECT flight_id,
       actual_arrival - scheduled_arrival AS arrival_delay
FROM flights
WHERE status = 'Arrived'
  AND actual_arrival IS NOT NULL
LIMIT 5;
```

**Шаг 2.** Тот же смысл в минутах – `extract(epoch FROM …) / 60`:

```sql
SELECT flight_id,
       extract(epoch FROM actual_arrival - scheduled_arrival) / 60 AS delay_minutes
FROM flights
WHERE status = 'Arrived'
  AND actual_arrival IS NOT NULL
LIMIT 5;
```

`epoch` – длина интервала в секундах; делим на 60. Отрицательное значение – рейс прибыл **раньше** плана.

**Шаг 3.** Средняя задержка по **всем** прибывшим рейсам – один агрегат, без `GROUP BY`:

```sql
SELECT round(avg(extract(epoch FROM actual_arrival - scheduled_arrival) / 60), 2) AS avg_delay_minutes
FROM flights
WHERE status = 'Arrived'
  AND actual_arrival IS NOT NULL;
```

**Шаг 4.** То же выражение внутри группы – например по `status` (групп мало, видно суть):

```sql
SELECT status,
       round(avg(extract(epoch FROM actual_arrival - scheduled_arrival) / 60), 2) AS avg_delay_minutes
FROM flights
WHERE actual_arrival IS NOT NULL
GROUP BY status
ORDER BY status;
```

Grain: один статус и средняя задержка по рейсам в этом статусе. Когда нужны **несколько** счётчиков в одной группе (сколько прилетело, сколько опоздало, среднее) – к `avg` добавляют `count(*)` и `FILTER`, как в части X; выражение для минут остаётся тем же.

---

## **Часть XII.** `string_agg` **– склеить значения группы**

> `string_agg` – агрегат: склеивает текст из строк группы в **одну** строку. Синтаксис: `string_agg(колонка, 'разделитель')`.

**Шаг 1.** Минимальный вариант – список классов тарифа по билету:

```sql
SELECT ticket_no,
       string_agg(fare_conditions, ', ') AS fare_list
FROM segments
WHERE flight_id <= 500
GROUP BY ticket_no
ORDER BY ticket_no
LIMIT 10;
```

Grain: одна строка – один `ticket_no`, в ячейке – все значения `fare_conditions` через запятую. Если у билета несколько сегментов с одним классом, класс может повториться в списке.

**Шаг 2.** Уникальные значения и порядок **внутри** строки:

```sql
SELECT ticket_no,
       string_agg(DISTINCT fare_conditions, ', ' ORDER BY fare_conditions) AS fare_classes
FROM segments
WHERE flight_id <= 500
GROUP BY ticket_no
ORDER BY ticket_no
LIMIT 10;
```

- `DISTINCT` внутри – не дублировать класс в списке.
- `ORDER BY` здесь относится к **склейке**, не к порядку строк результата.

Рядом с `string_agg` часто ставят другие агрегаты – например `sum(price)` – и отбирают группы через `HAVING`, как в любых групповых запросах выше.

---

## **Часть XIII. Подзапрос во** `FROM`

Иногда сначала нужна **промежуточная таблица**: сгруппировать, а потом ещё раз агрегировать.

```sql
SELECT round(avg(total_price), 2) AS avg_ticket_price,
       min(total_price) AS min_ticket_price,
       max(total_price) AS max_ticket_price
FROM (
    SELECT ticket_no, sum(price) AS total_price
    FROM segments
    WHERE flight_id <= 5000
    GROUP BY ticket_no
) ticket_totals;
```

Внутренний запрос – подзапрос во `FROM`. Ему **обязательно** нужен alias (`ticket_totals`). Снаружи grain – одна строка со статистикой по суммам билетов.

Логика та же, что у CTE в следующей части, только без имени на верхнем уровне. Порядок чтения: сначала внутренний блок, потом внешний.

---

## **Часть XIV.** `WITH` **– именованный шаг (CTE)**

> **CTE** (Common Table Expression) – подзапрос с именем после `WITH`, на который ссылается основной запрос.

```sql
WITH flight_revenue AS (
    SELECT flight_id, sum(price) AS revenue
    FROM segments
    WHERE flight_id <= 5000
    GROUP BY flight_id
)
SELECT flight_id, revenue
FROM flight_revenue
WHERE revenue > (SELECT avg(revenue) FROM flight_revenue)
ORDER BY revenue DESC, flight_id;
```

`flight_revenue` читается как временная таблица в рамках **одного** запроса. На следующей паре разберём CTE глубже – в связке с `JOIN`.

Зачем имя, если можно вложить подзапрос во `FROM`? Когда шагов несколько или финальный `SELECT` длинный – CTE держит мысль по этапам, как ячейки в ноутбуке.

---

## **Часть XV. Типичные ошибки**


| Ошибка                                                            | Почему ломается                                 |
| ----------------------------------------------------------------- | ----------------------------------------------- |
| Колонка в `SELECT` не в `GROUP BY` и не в агрегате                | неясно, какое значение из группы показать       |
| `WHERE avg(price) > 100`                                          | агрегата до группировки ещё нет                 |
| `COUNT(actual_arrival)` вместо `COUNT(*)`, когда нужны все строки | занижение счётчика из-за NULL                   |
| `round(price, 2)` до `avg`                                        | другое среднее                                  |
| Scalar-подзапрос возвращает много строк                           | ошибка выполнения                               |
| `LIMIT` без `ORDER BY` в top-N                                    | случайный срез                                  |
| `now()` вместо `bookings.now()`                                   | другая опорная дата на demo                     |
| Путаница grain                                                    | «одна строка – билет» vs «одна строка – статус» |


Перед сдачей HW2 для каждой задачи одной фразой: **одна строка результата – это…**

---

## **Another one шпаргалка!**


| Задача                | Приём                          | Пример                         |
| --------------------- | ------------------------------ | ------------------------------ |
| Сжать всю таблицу     | агрегат без `GROUP BY`         | `SELECT count(*) FROM flights` |
| Сжать по категории    | `GROUP BY`                     | `GROUP BY status`              |
| Фильтр строк          | `WHERE`                        | `WHERE status = 'Arrived'`     |
| Фильтр групп          | `HAVING`                       | `HAVING count(*) >= 10`        |
| Край дат в таблице    | `MIN` / `MAX`                  | ранний и поздний вылет         |
| Край внутри группы    | `MAX` в `GROUP BY`             | последний вылет в статусе      |
| Порог из таблицы      | scalar в `WHERE` / `HAVING`    | `> (SELECT avg(price) ...)`    |
| Уникальные в группе   | `COUNT(DISTINCT col)`          | билеты на рейсе                |
| Условный счёт         | `FILTER (WHERE ...)`           | вылетевшие в статусе           |
| Промежуточная таблица | подзапрос во `FROM` или `WITH` | сумма на билет, потом `avg`    |
| Список в ячейке       | `string_agg`                   | классы тарифа билета           |
| Задержка в минутах    | `extract(epoch FROM ...) / 60` | средняя задержка по аэропорту  |


Логический порядок:

```text
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

---

## **Часть XVI. Перед HW2**

Домашка проверяет не «знание синтаксиса», а **контракт результата**: grain, имена колонок, порядок строк, границы и округление. Ниже – карта «какой приём где», без готовых ответов.


| Задачи             | Что отработать из конспекта                      |
| ------------------ | ------------------------------------------------ |
| Q01, Q13           | `GROUP BY`, сортировка по агрегату               |
| Q02, Q11           | несколько агрегатов, `round(avg(...), 2)`        |
| Q03, Q06, Q10, Q14 | `HAVING`, порог на размер группы                 |
| Q04                | `MIN`/`MAX`, одна строка без `GROUP BY`          |
| Q12                | `MAX` внутри `GROUP BY`                          |
| Q05, Q09           | scalar-подзапрос в `WHERE` / `HAVING`            |
| Q07                | `WHERE` до группировки + `bookings.now()`        |
| Q08, Q16           | `FILTER`, задержка через `extract`               |
| Q15                | подзапрос во `FROM`, затем `avg` / `min` / `max` |
| Q17                | `string_agg`, `HAVING` на `sum(price)`           |
| Q18                | `WITH`, scalar по CTE                            |
| Q19                | `COUNT(DISTINCT ...)` внутри `GROUP BY`          |


**Чеклист перед отправкой файла:**

- [ ] имя файла из почты: `avivanov_hw2.sql`;
- [ ] маркеры `-- >>> Q01` … на месте, в блоке один statement;
- [ ] для каждой задачи проговорен grain;
- [ ] `LIMIT` только там, где в условии; после полного `ORDER BY` с tie-breaker;
- [ ] `bookings.now()` вместо `now()`;
- [ ] нет `SET search_path` в сдаваемом SQL.

---

## **Итог**

Сегодня мы:

- увидели, как SQL переходит от отдельных строк к показателям;
- освоили `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` и различие `COUNT(*)` с `COUNT(column)`;
- сгруппировали данные через `GROUP BY` и проговорили новый grain;
- развели `WHERE` и `HAVING` по этапам конвейера;
- нашли края набора через `MIN` и `MAX`;
- применили scalar-подзапрос, `COUNT(DISTINCT)`, `FILTER`;
- перевели задержку в минуты через `extract`;
- склеили текст группы через `string_agg`, затем собрали промежуточный результат во `FROM`;
- познакомились с `WITH` как именованным шагом.

На следующей паре соединим таблицы через `JOIN` и разберём CTE в многошаговых запросах.

## **Полезное**

Документация [Postgres Pro](https://postgrespro.ru/docs/postgresql/current/) – основной справочник курса.

- [Агрегатные функции](https://postgrespro.ru/docs/postgresql/current/functions-aggregate)
- [Выражения](https://postgrespro.ru/docs/postgresql/current/queries-table-expressions#QUERIES-GROUP) `GROUP BY` [и](https://postgrespro.ru/docs/postgresql/current/queries-table-expressions#QUERIES-GROUP) `HAVING`
- [Списки](https://postgrespro.ru/docs/postgresql/current/queries-select-lists) `SELECT`
- [Подзапросы](https://postgrespro.ru/docs/postgresql/current/functions-subquery)
- [FILTER](https://postgrespro.ru/docs/postgresql/current/sql-expressions#SYNTAX-AGGREGATES)
- [string_agg](https://postgrespro.ru/docs/postgresql/current/functions-aggregate#FUNCTIONS-AGGREGATE-TABLE)
- [CTE WITH](https://postgrespro.ru/docs/postgresql/current/queries-with)
- [Демобаза «Авиаперевозки»](https://postgrespro.ru/docs/postgrespro/current/demodb-bookings)

