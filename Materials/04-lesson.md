# **Lesson 4. Оконные функции**

На прошлых парах мы фильтровали строки, сжимали их через `GROUP BY` и соединяли таблицы. Сегодня научимся сравнивать строку с другими строками, не теряя её из результата.

---

## **Сегодня пройдемся по этому плану:**

1. Почему `GROUP BY` не решает все аналитические задачи.
2. `OVER` – вычислить функцию *поверх* набора строк.
3. `PARTITION BY` и `ORDER BY` в конце запроса.
4. Три семейства: Aggregate, Ranking, Value.
5. Агрегаты: накопительная сумма, скользящее среднее.
6. Рамка: `ROWS`, затем `RANGE`, затем `GROUPS`.
7. Ranking: место в порядке, Top-N, `NTILE`, доли.
8. Value: соседняя, первая и последняя строка.

---



## **Часть 0. Почему не хватит** `GROUP BY`

Сначала посмотрим на маленькие исходные данные:

```sql
SELECT airport_code,
       report_date,
       flight_count,
       revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

```text
DME | 2026-09-01 | 5 | 12000.00
DME | 2026-09-02 | 6 | 12000.00
DME | 2026-09-03 | 8 | 20000.00
SVO | 2026-09-01 | 7 | 10000.00
SVO | 2026-09-02 | 9 | 15000.00
SVO | 2026-09-03 | 8 | 15000.00
SVO | 2026-09-04 | 4 |  9000.00
VKO | 2026-09-01 | 3 |  8000.00
VKO | 2026-09-02 | 4 | 11000.00
VKO | 2026-09-03 | 6 | 14000.00
```

Одна строка – показатели одного аэропорта за одну дату.

Посчитаем общую выручку каждого аэропорта:

```sql
SELECT airport_code,
       sum(revenue) AS airport_revenue
FROM course.daily_airport_metrics
GROUP BY airport_code
ORDER BY airport_code;
```

```text
DME | 44000.00
SVO | 49000.00
VKO | 33000.00
```

Получили по одной строке на аэропорт. Отдельные даты исчезли.

Но что делать, если мы хотим видеть каждый день и рядом итог аэропорта? Здесь нам и понадобится оконная функция.

> **Оконная функция** – вычисляет значение по набору строк, связанному с текущей строкой, но не сжимает эти строки в одну.

```text
GROUP BY → одна строка на группу
OVER     → исходные строки остаются
```

---



## **Часть I.** `OVER` **– поверх набора**

Английское `OVER` буквально значит «поверх». Синтаксис читается так:

```text
func(...) OVER (...)
```

«вычисли `func(...)` *поверх* набора строк». Функция, ясное дело, к чему-то применяется.

К примеру, для вычисления суммы мы будем использовать:

```text
sum(...) OVER (...)
```

Набор в скобках после `OVER` – это и есть окно. Пока скобки пустые, набор – все строки результата:

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER () AS total_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Первые строки результата:

```text
DME | 2026-09-01 | 12000.00 | 126000.00
DME | 2026-09-02 | 12000.00 | 126000.00
DME | 2026-09-03 | 20000.00 | 126000.00
```

Каждая дата осталась отдельной строкой. `126000` – сумма всех десяти строк – повторяется рядом с каждой датой.

Без `OVER` агрегат `sum(revenue)` сжал бы строки. С `OVER` тот же агрегат считается *поверх* набора, а строки на месте.

---



## **Часть II. Делим набор через** `PARTITION BY`

Теперь нужен отдельный итог для каждого аэропорта, для этого мы познакомимся с партициями:

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
       ) AS airport_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

```text
DME | 2026-09-01 | 12000.00 | 44000.00
DME | 2026-09-02 | 12000.00 | 44000.00
DME | 2026-09-03 | 20000.00 | 44000.00
SVO | 2026-09-01 | 10000.00 | 49000.00
...
```

> `PARTITION BY` режет результат на независимые партиции. Функция считается заново *поверх* каждой партиции.

Партиции не пересекаются и вместе покрывают **все строки результата**. Каждая строка попадает ровно в одну партицию. Дыр нет: нет строки «ни в одном аэропорте» и нет строки сразу в `DME` и в `SVO`.

Нет `PARTITION BY` – весь результат считается одной партицией. Именно так работал `OVER ()`.

```text
OVER ()                          → одна сумма 126000 для всех строк
OVER (PARTITION BY airport_code) → своя сумма 44000 / 49000 / 33000
```

Одна строка результата по-прежнему означает один аэропорт в одну дату.

---



## **Часть III.** `ORDER BY` **в конце запроса**

В оконном запросе легко перепутать две сортировки.

`ORDER BY` **в конце запроса** печатает уже посчитанные строки. Он не участвует в расчёте окна:

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
       ) AS airport_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Поменяем только хвост на `ORDER BY revenue DESC` – числа `airport_revenue` останутся теми же, изменится лишь порядок на экране.

Внутри `OVER` тоже можно написать `ORDER BY`. Это уже не про печать, а про логический порядок строк *внутри партиции*: кто предыдущий, кто следующий, в каком порядке копить сумму.

Если итоговый порядок важен, его всегда задаём `ORDER BY` в конце запроса. На внутреннюю сортировку окна полагаться нельзя.

К `ORDER BY` внутри `OVER` вернёмся, когда сумма должна расти по датам.

---



## **Часть IV. Три семейства оконных функций**

Окно одно и то же: «посчитай *поверх* набора». А функции над этим набором бывают трёх семейств.


| Aggregate                   | Ranking                                    | Value                                          |
| --------------------------- | ------------------------------------------ | ---------------------------------------------- |
| `SUM()`, `AVG()`, `COUNT()` | `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`   | `LAG()`, `LEAD()`                              |
| `MIN()`, `MAX()`            | `NTILE()`, `CUME_DIST()`, `PERCENT_RANK()` | `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()` |


- **Aggregate** – одно число на набор: сумма, среднее, минимум. Строки на месте, рядом появляется итог.
- **Ranking** – место строки в порядке: какой она по счёту, в какой корзине, какая доля до неё.
- **Value** – значение из другой строки набора: вчерашняя выручка, первая дата, последняя дата.

Дальше идём по семействам: сначала агрегаты, потом места, потом значения из соседей.

---



## **Часть V. Aggregate: итог поверх партиции**

Любой агрегат, который мы знаем по `GROUP BY`, можно посчитать и *поверх* окна: `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`. Разница одна: строки не сжимаются.

Вот так можно – несколько итогов аэропорта сразу, каждый день на месте:

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
       ) AS airport_revenue,
       round(
           avg(revenue) OVER (
               PARTITION BY airport_code
           ),
           2
       ) AS airport_avg,
       count(*) OVER (
           PARTITION BY airport_code
       ) AS day_count,
       min(revenue) OVER (
           PARTITION BY airport_code
       ) AS min_revenue,
       max(revenue) OVER (
           PARTITION BY airport_code
       ) AS max_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME`:

```text
дата       | revenue | sum    | avg     | count | min   | max
2026-09-01 | 12000   | 44000  | 14666.67| 3     | 12000 | 20000
2026-09-02 | 12000   | 44000  | 14666.67| 3     | 12000 | 20000
2026-09-03 | 20000   | 44000  | 14666.67| 3     | 12000 | 20000
```

Три даты остались тремя строками. Рядом одно и то же: сумма `44000`, среднее, число дней, минимум и максимум. Пока внутри `OVER` нет `ORDER BY`, агрегат видит **всю** партицию.

`FILTER` знаком нам по обычным агрегатам. Его тоже можно поставить к окну: строки все, в сумму – только подходящие.

```sql
SELECT airport_code,
       report_date,
       revenue,
       flight_count,
       sum(revenue) FILTER (
           WHERE flight_count >= 6
       ) OVER (
           PARTITION BY airport_code
       ) AS revenue_on_busy_days
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME` дни с шестью рейсами и больше – 2 и 3 сентября (`12000 + 20000`). 1 сентября в сумму не входит, но строка остаётся:

```text
дата       | revenue | flights | revenue_on_busy_days
2026-09-01 | 12000   | 5       | 32000
2026-09-02 | 12000   | 6       | 32000
2026-09-03 | 20000   | 8       | 32000
```

`FILTER` не меняет партицию: дни `DME` по-прежнему считаются вместе, просто часть из них не входит в сумму.

Если нужно разложить строки **по условию в разные партиции**, в `PARTITION BY` можно поставить `CASE`. Это тот же `CASE`, что на первой паре: выражение, не колонка.

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY CASE
               WHEN revenue >= 15000 THEN 'high'
               ELSE 'other'
           END
       ) AS revenue_by_size
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Строки с выручкой `15000` и выше попадают в одну партицию `high`, остальные – в `other`. Аэропорт больше не режет набор.

`high`: `20000 + 15000 + 15000 = 50000`. `other`: остальные семь дней, `76000`.

Для `DME`:

```text
дата       | revenue | revenue_by_size
2026-09-01 | 12000   | 76000
2026-09-02 | 12000   | 76000
2026-09-03 | 20000   | 50000
```

3 сентября оказалось в одной партиции с двумя днями `SVO` по `15000`, а не с соседними датами `DME`.

Так можно: `PARTITION BY` принимает любое выражение, в том числе `CASE`. Одинаковый результат выражения – одна партиция.

---



## **Часть VI. Aggregate: накопительный итог**

Руководителю нужна сумма с начала месяца до текущей даты, а не сразу итог аэропорта. Для этого внутри `OVER` появляется `ORDER BY report_date`: функция идёт по датам.

И явно говорим, какие строки брать: от первой до текущей.

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND CURRENT ROW
       ) AS running_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME` набор растёт по одной дате:

```text
2026-09-01 | 12000.00 | 12000.00
2026-09-02 | 12000.00 | 24000.00
2026-09-03 | 20000.00 | 44000.00
```

> **Рамка** – те строки партиции, которые агрегат реально видит для *этой* текущей строки. То же самое по-английски называют **frame**, фрейм. Рамка и фрейм – одно слово.

Границы рамки читаются относительно текущей строки:

```text
PRECEDING             → назад по ленте, «до текущей»
CURRENT ROW           → текущая строка
FOLLOWING             → вперёд по ленте, «после текущей»
UNBOUNDED PRECEDING   → от самого начала партиции
UNBOUNDED FOLLOWING   → до самого конца партиции
```

`BETWEEN ... AND ...` задаёт начало и конец. В накопительном итоге начало – `UNBOUNDED PRECEDING` (первая строка партиции), конец – `CURRENT ROW` (текущая строка).

`ROWS` считает физические строки ленты: одна строка, вторая, третья. Пока работаем только с `ROWS`. Другие способы отсчитать «до» и «после» появятся после скользящего среднего.

---



## **Часть VII. Aggregate: скользящее среднее**

Посчитаем среднюю выручку текущей строки и двух предыдущих:

```sql
SELECT airport_code,
       report_date,
       revenue,
       round(
           avg(revenue) OVER (
               PARTITION BY airport_code
               ORDER BY report_date
               ROWS BETWEEN 2 PRECEDING
                        AND CURRENT ROW
           ),
           2
       ) AS moving_avg_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME`:

```text
2026-09-01 | 12000.00 | 12000.00
2026-09-02 | 12000.00 | 12000.00
2026-09-03 | 20000.00 | 14666.67
```

У края партиции строк меньше, чем просили: для первой даты доступна только она сама, для второй – две. Начиная с третьей – максимум три.

`2 PRECEDING` здесь – две **строки** назад по ленте `ORDER BY report_date`, а не «два календарных дня». Если бы в таблице пропустили дату, `ROWS` всё равно взял бы две соседние строки.

`FOLLOWING` пока не используем: среднее смотрит назад и на текущий день, не вперёд.

---



## **Часть VIII. Рамка: строки, значения, группы**

Агрегат смотрит не на всю партицию, а на рамку текущей строки. Партиция общая, рамка у каждой строки своя.

```text
результат запроса
    └── партиция          ← PARTITION BY
            └── лента     ← ORDER BY внутри OVER
                    └── рамка текущей строки
```

> **Партиция** – независимый кусок результата. Вместе партиции покрывают все строки, без пересечений.

> **Логический порядок** – лента внутри партиции. Задаётся `ORDER BY` внутри `OVER`. Это не `ORDER BY` в конце запроса.

> **Окно** – вся спецификация в `OVER (...)`.

> **Рамка** – какой отрезок ленты видит функция для текущей строки. Рамка и фрейм – одно и то же: `frame` в документации PostgreSQL, «рамка» в этом конспекте.

Партиция – это `DME` отдельно, `SVO` отдельно, `VKO` отдельно. Рамка живёт **внутри одной партиции**. Соседние строки, значения и группы никогда не берутся из другого аэропорта.

Начало и конец рамки:

```text
N PRECEDING           → на N единиц назад
CURRENT ROW           → текущая единица
N FOLLOWING           → на N единиц вперёд
UNBOUNDED PRECEDING   → от начала этой партиции
UNBOUNDED FOLLOWING   → до конца этой партиции
```

`FOLLOWING` – это «после текущей» по ленте `ORDER BY` внутри `OVER`. В `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` читается так: одна строка до, текущая, одна строка после.

Возьмём четыре дня **одного** аэропорта `SVO` – на них дальше все примеры этой части. Другие партиции в расчёт не входят:

```text
дата       | выручка
2026-09-01 | 10000
2026-09-02 | 15000
2026-09-03 | 15000
2026-09-04 |  9000
```

Одна строка по-прежнему означает один аэропорт в одну дату.

### **Peer и peer-группа**

Выстроим `SVO` по выручке:

```text
логическое место:  1      2       3       4
дата:              04     01      02      03
выручка:           9000   10000   15000   15000
```

> **Peer** – строка, у которой значения колонок оконного `ORDER BY` совпадают с текущей.

> **Peer-группа** – все подряд идущие peer-ы с одним значением `ORDER BY` **внутри одной партиции**.

При `ORDER BY revenue` дни 2 и 3 сентября – peer-ы: оба `15000`. Они образуют одну peer-группу.

При `ORDER BY report_date` peer-ов нет: даты внутри аэропорта не повторяются.

Peer считается только по оконному `ORDER BY` **внутри своей партиции**. Сменили колонку сортировки – другие peer-ы. Партиция при этом та же: всё ещё `SVO`. Строка `DME` с выручкой `12000` в эту ленту не входит.

### `ROWS` **– соседние строки**

Мы уже писали `ROWS`: единица шага – физическая строка ленты.

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY revenue, report_date
           ROWS BETWEEN 1 PRECEDING
                    AND 1 FOLLOWING
       ) AS sum_adj_rows
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY revenue, report_date;
```

```text
дата       | revenue | sum_adj_rows | какие строки
2026-09-04 |  9000   | 19000        | 9000 + 10000
2026-09-01 | 10000   | 34000        | 9000 + 10000 + 15000
2026-09-02 | 15000   | 40000        | 10000 + 15000 + 15000
2026-09-03 | 15000   | 30000        | 15000 + 15000
```

`1 PRECEDING` – одна строка **до** текущей. `1 FOLLOWING` – одна строка **после** текущей. Вместе с текущей это три места на ленте `9000, 10000, 15000, 15000`. Сосед определяется местом на ленте, не величиной выручки. У края ленты «после» может не быть: 3 сентября `FOLLOWING` пустой, поэтому в сумме только `15000 + 15000`.

`report_date` в `ORDER BY` нужен, чтобы две строки с `15000` стояли в стабильном порядке.

### `RANGE` **– соседние значения**

`RANGE` считает не строки, а значения колонки оконного `ORDER BY`.

Сначала календарный диапазон. Даты внутри `SVO` уникальны, поэтому peer-ов нет:

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
           RANGE BETWEEN INTERVAL '1 day' PRECEDING
                     AND CURRENT ROW
       ) AS revenue_two_days
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY report_date;
```

```text
дата       | revenue | revenue_two_days
2026-09-01 | 10000   | 10000
2026-09-02 | 15000   | 25000
2026-09-03 | 15000   | 30000
2026-09-04 |  9000   | 24000
```

`INTERVAL '1 day' PRECEDING` означает: взять строки, у которых дата лежит в отрезке `[текущая дата - 1 день, текущая дата]`.

Здесь числа совпадут с `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW`: даты идут подряд и не повторяются. Без дубликатов `ORDER BY` режимы часто неотличимы. Поэтому peer-ы так важны.

Теперь диапазон по выручке. Числовое смещение – это не «N строк», а «N единиц значения»:

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY revenue
           RANGE BETWEEN 5000 PRECEDING
                     AND CURRENT ROW
       ) AS sum_range_5000
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY revenue, report_date;
```

```text
дата       | revenue | sum_range_5000 | диапазон значений
2026-09-04 |  9000   |  9000          | [4000,  9000]
2026-09-01 | 10000   | 19000          | [5000, 10000]
2026-09-02 | 15000   | 40000          | [10000, 15000]
2026-09-03 | 15000   | 40000          | [10000, 15000]
```

Для обоих дней с `15000` рамка одна и та же: выручка от `10000` до `15000`. День с `9000` в неё не входит: `15000 - 9000 = 6000`, это больше смещения `5000`.

Для `RANGE` с числовым или интервальным смещением в `ORDER BY` должна быть ровно одна колонка, и она числовая или временная. `ROWS` допускает несколько колонок.

Смещение `RANGE BETWEEN 1 PRECEDING` по выручке означало бы «1 рубль», а не «одна соседняя строка». Единицу смещения читаем из типа колонки `ORDER BY`.

### `GROUPS` **– соседние peer-группы**

`GROUPS` считает не строки и не рубли, а **группы равных значений** `ORDER BY`.

Это группы внутри **одной партиции**. Здесь партиция – аэропорт `SVO`. Группа `15000` у `SVO` не склеивается с `12000` у `DME` и вообще не видит другие аэропорты. Сравниваются соседние peer-группы на ленте одного `SVO`, а не разные партиции между собой.

При `ORDER BY revenue` у `SVO` три группы:

```text
внутри партиции SVO:
G1  9000           → 4 сентября
G2  10000          → 1 сентября
G3  15000, 15000   → 2 и 3 сентября
```

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY revenue
           GROUPS BETWEEN 1 PRECEDING
                      AND 1 FOLLOWING
       ) AS sum_adj_groups
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY revenue, report_date;
```

```text
дата       | revenue | sum_adj_groups | какие группы
2026-09-04 |  9000   | 19000          | G1 + G2
2026-09-01 | 10000   | 49000          | G1 + G2 + G3
2026-09-02 | 15000   | 40000          | G2 + G3
2026-09-03 | 15000   | 40000          | G2 + G3
```

`1 PRECEDING` / `1 FOLLOWING` здесь – одна **группа** до и одна после, снова внутри той же партиции `SVO`. Внутри группы G3 обе строки видят одну и ту же рамку: `10000 + 15000 + 15000 = 40000`.

Сравните с `ROWS` выше: там 2 и 3 сентября получили разные суммы `40000` и `30000`, потому что считались физические соседи. В `GROUPS` они неразличимы.

### **Что такое** `CURRENT ROW`


| Режим    | Единица шага            | `CURRENT ROW` – это          |
| -------- | ----------------------- | ---------------------------- |
| `ROWS`   | физическая строка ленты | одна текущая строка          |
| `RANGE`  | значение `ORDER BY`     | все peer-ы текущего значения |
| `GROUPS` | peer-группа             | вся текущая peer-группа      |


`PRECEDING` и `FOLLOWING` отсчитываются в той же единице. Без оконного `ORDER BY` единиц нет: нет «до» и «после».

`ORDER BY` в конце запроса к рамке не относится. Он только печатает уже посчитанные строки.

Полный синтаксис рамки, который мы теперь можем прочитать целиком:

```text
{ROWS | RANGE | GROUPS} BETWEEN ... AND ...
```



### **Какую рамку PostgreSQL ставит сам**

Сначала простой случай. Возьмём только `DME` и напишем оконный `ORDER BY`, но рамку не укажем:

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS running_without_frame
FROM course.daily_airport_metrics
WHERE airport_code = 'DME'
ORDER BY report_date;
```

```text
дата       | revenue | running_without_frame
2026-09-01 | 12000   | 12000
2026-09-02 | 12000   | 24000
2026-09-03 | 20000   | 44000
```

Мы не писали `BETWEEN`, а сумма всё равно растёт от первой даты к текущей – как накопительный итог из части VI. Значит, PostgreSQL сам подставил рамку «от начала партиции до текущей».

Сортировка здесь по `report_date`, не по выручке. Два дня с `12000` не peer-ы: даты разные. Поэтому «до текущей даты» и «до текущей строки» совпадают. Разницы с явным `ROWS` ещё не видно.

Теперь тот же приём на `SVO`, но лента по выручке, где два дня с `15000`:

```sql
SELECT report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY revenue
       ) AS default_frame
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY revenue, report_date;
```

```text
дата       | revenue | default_frame
2026-09-04 |  9000   |  9000
2026-09-01 | 10000   | 19000
2026-09-02 | 15000   | 49000
2026-09-03 | 15000   | 49000
```

2 сентября уже `49000`, хотя «следующий» день с той же выручкой как будто ещё впереди. Так бывает, потому что default – это не `ROWS`, а `RANGE`: `CURRENT ROW` захватывает все peer-ы текущего значения. Оба `15000` входят сразу.

PostgreSQL подставляет вот это:

```text
нет ORDER BY в OVER  →  вся партиция
есть ORDER BY        →  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

Явный `ROWS ... CURRENT ROW` на той же ленте дал бы 2 сентября только `34000`: `9000 + 10000 + 15000`, без 3 сентября. Для предсказуемого построчного накопления рамку пишем явно через `ROWS`.

### **Исключаем строки из рамки**

Иногда из уже выбранной рамки нужно убрать текущую строку или её группу.

```sql
SELECT airport_code,
       report_date,
       revenue,
       sum(revenue) OVER (
           PARTITION BY airport_code
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
           EXCLUDE CURRENT ROW
       ) AS other_days_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для 1 сентября `DME`: `44000 - 12000 = 32000`. Рамка – вся партиция этого аэропорта, затем из неё выкинули текущую строку.

```text
DME | 2026-09-01 | 12000 | 32000
DME | 2026-09-02 | 12000 | 32000
DME | 2026-09-03 | 20000 | 24000
```


| Вариант               | Что убирает                          |
| --------------------- | ------------------------------------ |
| `EXCLUDE CURRENT ROW` | только текущую строку                |
| `EXCLUDE GROUP`       | всю peer-группу текущей строки       |
| `EXCLUDE TIES`        | peer-ов, но текущую строку оставляет |
| `EXCLUDE NO OTHERS`   | ничего; так и работает по умолчанию  |


`EXCLUDE` не выбирает рамку заново. Он только вычитает строки из уже заданных `BETWEEN ... AND ...`.

Агрегатам рамка нужна. Некоторым другим оконным функциям хватает порядка на ленте. Это будет видно в семействах Ranking и Value.

---



## **Часть IX. Ranking: место в порядке**

Семейство Ranking отвечает на вопрос «какое место у этой строки на ленте».

Сравним три функции на одинаковых значениях выручки:

```sql
SELECT airport_code,
       report_date,
       revenue,
       row_number() OVER (
           PARTITION BY airport_code
           ORDER BY revenue DESC, report_date
       ) AS row_number,
       rank() OVER (
           PARTITION BY airport_code
           ORDER BY revenue DESC
       ) AS rank,
       dense_rank() OVER (
           PARTITION BY airport_code
           ORDER BY revenue DESC
       ) AS dense_rank
FROM course.daily_airport_metrics
ORDER BY airport_code, revenue DESC, report_date;
```

Посмотрим на `SVO`:

```text
date       | revenue | row_number | rank | dense_rank
-----------+---------+------------+------+-----------
2026-09-02 | 15000   | 1          | 1    | 1
2026-09-03 | 15000   | 2          | 1    | 1
2026-09-01 | 10000   | 3          | 3    | 2
2026-09-04 |  9000   | 4          | 4    | 3
```

> `ROW_NUMBER()` выдаёт каждой строке уникальный последовательный номер.

> `RANK()` выдаёт одинаковым значениям одинаковый ранг и оставляет пропуск после ничьей.

> `DENSE_RANK()` выдаёт одинаковым значениям одинаковый ранг, но не оставляет пропусков.

Две строки с `15000` – те самые peer-ы по `ORDER BY revenue`. `RANK` и `DENSE_RANK` дают им одно место. `ROW_NUMBER` всё равно нумерует по одной: для него добавлен tie-breaker `report_date`.

Этим функциям рамка не нужна. Им достаточно партиции и ленты `ORDER BY`.

Если итоговый порядок на экране важен, снова пишем `ORDER BY` в конце запроса.

### **Top-N внутри каждой группы**

Найдём два самых доходных дня каждого аэропорта. Сначала нумеруем, потом оставляем номера 1 и 2.

Оконную функцию нельзя использовать в `WHERE` того же уровня: `WHERE` выполняется раньше окна. Поэтому номер считаем в CTE, затем фильтруем:

```sql
WITH ranked_days AS (
    SELECT airport_code,
           report_date,
           revenue,
           row_number() OVER (
               PARTITION BY airport_code
               ORDER BY revenue DESC, report_date
           ) AS revenue_number
    FROM course.daily_airport_metrics
)
SELECT airport_code,
       report_date,
       revenue
FROM ranked_days
WHERE revenue_number <= 2
ORDER BY airport_code, revenue DESC, report_date;
```

```text
DME | 2026-09-03 | 20000.00
DME | 2026-09-01 | 12000.00
SVO | 2026-09-02 | 15000.00
SVO | 2026-09-03 | 15000.00
VKO | 2026-09-03 | 14000.00
VKO | 2026-09-02 | 11000.00
```



### `NTILE` **– разрезать очередь на n кусков**

Выстроим дни `SVO` от большой выручки к маленькой. Получилась очередь из четырёх строк. `NTILE(2)` режет эту очередь на две группы почти поровну: первые две строки – группа 1, вторые две – группа 2.

```sql
SELECT airport_code,
       report_date,
       revenue,
       ntile(2) OVER (
           PARTITION BY airport_code
           ORDER BY revenue DESC, report_date
       ) AS revenue_half
FROM course.daily_airport_metrics
ORDER BY airport_code, revenue_half, revenue DESC, report_date;
```

Для `SVO`:

```text
2026-09-02 | 15000.00 | 1
2026-09-03 | 15000.00 | 1
2026-09-01 | 10000.00 | 2
2026-09-04 |  9000.00 | 2
```

Это не порог в рублях. Мы не спрашивали «кто заработал больше 12000». Мы разрезали упорядоченный список пополам по числу строк.

Если строк не делится нацело, первые группы будут на одну строку длиннее.

### `PERCENT_RANK` **и** `CUME_DIST` **– где я на линейке**

Оставим четыре дня `SVO` и выстроим их по выручке: `9000`, `10000`, `15000`, `15000`.

`PERCENT_RANK` показывает относительное положение не самой строки, а её ранга. Для партиции из `N` строк он считается так: `(RANK - 1) / (N - 1)`.

Строки одной peer-группы получают одинаковый ранг и одинаковый `PERCENT_RANK`. Первая peer-группа всегда получает `0`. Если последняя peer-группа состоит из `K` строк, её ранг равен `N - K + 1`, а `PERCENT_RANK` – `(N - K) / (N - 1)`. Поэтому значение `1` получится только тогда, когда в последней peer-группе одна строка.

> **Примечание.** Для партиции из одной строки PostgreSQL возвращает `0`.

В нашем примере две строки с `15000` делят третий ранг, поэтому обе получают `(3 - 1) / (4 - 1)`, то есть примерно `0.67`.

`CUME_DIST` показывает, какая доля партиции находится от начала порядка до конца текущей peer-группы включительно. Для `15000` это все 4 строки из 4, то есть `1`.

> **Примечание.** При сортировке по возрастанию это доля строк со значением не больше текущего. При сортировке по убыванию – не меньше текущего.

```sql
SELECT report_date,
       revenue,
       round(
           percent_rank() OVER (
               PARTITION BY airport_code
               ORDER BY revenue
           )::numeric,
           2
       ) AS percent_rank,
       round(
           cume_dist() OVER (
               PARTITION BY airport_code
               ORDER BY revenue
           )::numeric,
           2
       ) AS cume_dist
FROM course.daily_airport_metrics
WHERE airport_code = 'SVO'
ORDER BY revenue, report_date;
```

```text
дата       | revenue | percent_rank | cume_dist
2026-09-04 |  9000   | 0.00         | 0.25
2026-09-01 | 10000   | 0.33         | 0.50
2026-09-02 | 15000   | 0.67         | 1.00
2026-09-03 | 15000   | 0.67         | 1.00
```

Коротко:

- `PERCENT_RANK` – как далеко я от начала линейки;
- `CUME_DIST` – какую долю очереди я уже закрыл, включая тех, кто рядом со мной.

Формула для проверки `CUME_DIST`: `(строки от начала партиции до конца текущей peer-группы) / (число строк партиции)`.

---



## **Часть X. Value: значение из другой строки**

Семейство Value не считает сумму и не выдаёт место. Оно приносит значение из другой строки набора.

### `LAG` **и** `LEAD`

> `LAG(value)` берёт значение из предыдущей строки по оконному `ORDER BY`.

Рамку `LAG` не читает: «предыдущая» – сосед по ленте.

```sql
SELECT airport_code,
       report_date,
       revenue,
       lag(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS previous_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME`:

```text
2026-09-01 | 12000.00 | NULL
2026-09-02 | 12000.00 | 12000.00
2026-09-03 | 20000.00 | 12000.00
```

Для первой даты аэропорта предыдущей строки нет, поэтому получаем `NULL`.

Добавим изменение:

```sql
SELECT airport_code,
       report_date,
       revenue,
       revenue - lag(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS revenue_change
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Изменение `DME`:

```text
2026-09-01 | 12000.00 | NULL
2026-09-02 | 12000.00 | 0.00
2026-09-03 | 20000.00 | 8000.00
```

> `LEAD(value)` работает в другую сторону и берёт значение из следующей строки.

```sql
SELECT airport_code,
       report_date,
       revenue,
       lead(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS next_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME`:

```text
2026-09-01 | 12000.00 | 12000.00
2026-09-02 | 12000.00 | 20000.00
2026-09-03 | 20000.00 | NULL
```

3 сентября следующей даты в партиции нет, поэтому `NULL`. `LEAD`, как и `LAG`, не ходит в `SVO` или `VKO`: следующая строка ищется только внутри своего аэропорта.

Полная форма `lag(value, offset, default)` позволяет изменить сдвиг и значение при отсутствии строки:

```sql
SELECT airport_code,
       report_date,
       revenue,
       lag(revenue, 2, 0) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS revenue_two_days_back
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME` сдвиг на две строки назад. Если такой строки нет, подставляем `0`:

```text
2026-09-01 | 12000.00 | 0.00
2026-09-02 | 12000.00 | 0.00
2026-09-03 | 20000.00 | 12000.00
```



### `FIRST_VALUE`**,** `LAST_VALUE`**,** `NTH_VALUE`

> `FIRST_VALUE(value)` возвращает значение из первой строки рамки.

> `LAST_VALUE(value)` возвращает значение из последней строки рамки, а не обязательно из последней строки всей партиции.

Сначала посмотрим, что будет, если рамку не написать. Мы уже знаем default: `RANGE ... CURRENT ROW`.

```sql
SELECT airport_code,
       report_date,
       revenue,
       last_value(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
       ) AS last_revenue_in_frame
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для `DME` даты уникальны, рамка по умолчанию заканчивается на текущей дате:

```text
2026-09-01 | 12000.00 | 12000.00
2026-09-02 | 12000.00 | 12000.00
2026-09-03 | 20000.00 | 20000.00
```

`last_revenue_in_frame` совпал с `revenue` текущей строки. «Последняя» оказалась текущей, потому что рамка ещё не дошла до конца партиции.

Если нужна последняя строка всей партиции, зададим обе границы явно. `UNBOUNDED FOLLOWING` как раз значит «до конца этой партиции»:

```sql
SELECT airport_code,
       report_date,
       revenue,
       first_value(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
       ) AS first_revenue,
       last_value(revenue) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
       ) AS last_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Для каждой строки `DME` одна и та же пара: первая дата партиции и последняя.

```text
2026-09-01 | 12000.00 | 12000.00 | 20000.00
2026-09-02 | 12000.00 | 12000.00 | 20000.00
2026-09-03 | 20000.00 | 12000.00 | 20000.00
```

`NTH_VALUE(value, n)` берёт значение из строки с номером `n` внутри рамки:

```sql
SELECT airport_code,
       report_date,
       nth_value(revenue, 2) OVER (
           PARTITION BY airport_code
           ORDER BY report_date
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
       ) AS second_revenue
FROM course.daily_airport_metrics
ORDER BY airport_code, report_date;
```

Вторая дата `DME` – 2 сентября, выручка `12000`. Рамка на всю партицию, поэтому это число стоит у всех трёх строк:

```text
2026-09-01 | 12000.00
2026-09-02 | 12000.00
2026-09-03 | 12000.00
```

PostgreSQL использует поведение `RESPECT NULLS`: `NULL` не пропускаются автоматически.

---



## **Часть XI. Именованное окно**

Когда несколько функций используют одну партицию и порядок, определение можно назвать:

```sql
SELECT airport_code,
       report_date,
       revenue,
       row_number() OVER airport_days AS day_number,
       lag(revenue) OVER airport_days AS previous_revenue
FROM course.daily_airport_metrics
WINDOW airport_days AS (
    PARTITION BY airport_code
    ORDER BY report_date
)
ORDER BY airport_code, report_date;
```

Именованное окно не меняет расчёт. Оно только убирает повторение одинакового `PARTITION BY` и `ORDER BY`.

---



## **Часть XII. Реальные вопросы в** `bookings`



### **Место цены внутри класса**

```sql
SELECT ticket_no,
       flight_id,
       fare_conditions,
       price,
       dense_rank() OVER (
           PARTITION BY fare_conditions
           ORDER BY price DESC
       ) AS price_rank
FROM segments
WHERE flight_id <= 5
ORDER BY fare_conditions, price_rank, ticket_no
LIMIT 20;
```



### **Предыдущий вылет из аэропорта**

```sql
SELECT flight_id,
       departure_airport,
       scheduled_departure,
       lag(scheduled_departure) OVER (
           PARTITION BY departure_airport
           ORDER BY scheduled_departure, flight_id
       ) AS previous_departure
FROM timetable
WHERE departure_airport = 'SVO'
  AND flight_id <= 100
ORDER BY scheduled_departure, flight_id
LIMIT 15;
```



### **Top-2 рейса на аэропорт**

```sql
WITH numbered_flights AS (
    SELECT flight_id,
           departure_airport,
           scheduled_departure,
           row_number() OVER (
               PARTITION BY departure_airport
               ORDER BY scheduled_departure DESC, flight_id DESC
           ) AS flight_number
    FROM timetable
    WHERE flight_id <= 20
)
SELECT flight_id,
       departure_airport,
       scheduled_departure
FROM numbered_flights
WHERE flight_number <= 2
ORDER BY departure_airport, scheduled_departure DESC, flight_id DESC;
```

---



## **Шпаргалка**


| Вопрос                            | Конструкция                                                  |
| --------------------------------- | ------------------------------------------------------------ |
| посчитать *поверх* набора         | `функция(...) OVER (...)`                                    |
| общий показатель на каждой строке | `aggregate(...) OVER ()`                                     |
| показатель внутри группы          | `OVER (PARTITION BY ...)`                                    |
| партиция по условию               | `PARTITION BY CASE WHEN ... END`                             |
| печать результата                 | `ORDER BY` в конце запроса                                   |
| рамка = фрейм                     | `{ROWS                                                       |
| назад / вперёд по ленте           | `PRECEDING` / `FOLLOWING`                                    |
| логический порядок внутри группы  | `ORDER BY` внутри `OVER`                                     |
| накопительный итог по строкам     | агрегат + `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |
| соседние строки                   | `ROWS BETWEEN N PRECEDING AND ...`                           |
| соседние значения                 | `RANGE BETWEEN ...`                                          |
| соседние peer-группы              | `GROUPS BETWEEN ...`                                         |
| уникальный номер                  | `ROW_NUMBER()`                                               |
| место с пропусками после ties     | `RANK()`                                                     |
| место без пропусков               | `DENSE_RANK()`                                               |
| разрезать очередь на n кусков     | `NTILE(n)`                                                   |
| где я на линейке                  | `PERCENT_RANK` / `CUME_DIST`                                 |
| предыдущее / следующее            | `LAG` / `LEAD`                                               |
| первая / последняя строка рамки   | `FIRST_VALUE` / `LAST_VALUE`                                 |


Домашнее задание: [HW 4](../HW/HW_4/README.md). Обязательная часть – Q01–Q14 на 20 баллов. Бонус Q15–Q19 – по 1 баллу, максимум 25.

Как создают таблицы, меняют строки и читают схему через `\d` – в [дополнительном конспекте](extra-lesson.md).

До встречи на следующей страничке!

## **Полезное**

- [Вызовы оконных функций](https://postgrespro.ru/docs/postgresql/current/sql-expressions#SYNTAX-WINDOW-FUNCTIONS)
- [Оконные функции](https://postgrespro.ru/docs/postgresql/current/functions-window)
- [Учебные таблицы](../Others/course_tables.md) `course`
- [Как смотрят схему и сохраняют результат](extra-lesson.md)

