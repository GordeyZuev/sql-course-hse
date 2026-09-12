# HW 3. JOIN и операции над множествами

16 задач, **20 баллов**. Q01–Q08 – по 1 баллу, Q09–Q16 – по 1,5 балла.

Сдавать файл **обязательно** с именем из почты `@edu.hse.ru`:

- `avivanov@edu.hse.ru` → `avivanov_hw3.sql`
- `avivanov_1@edu.hse.ru` → `avivanov_1_hw3.sql`

Сложность растёт постепенно:

1. Q01–Q08 – виды `JOIN` на маленьких таблицах `course.*`;
2. Q07 и Q09–Q11 – операции `UNION [ALL]`, `INTERSECT`, `EXCEPT`;
3. Q12–Q16 – применение тех же идей к большой схеме `bookings`.

**Что проверяет чекер:**

- Q01 – `JOIN`;
- Q02 – `USING`;
- Q03 – `LEFT JOIN`;
- Q04 – `RIGHT JOIN`;
- Q05 – `FULL JOIN`;
- Q06 – `LEFT JOIN` и `IS NULL`;
- Q07 – `UNION` без `ALL`;
- Q08 – `CROSS JOIN`;
- Q09 – `UNION ALL`;
- Q10 – `INTERSECT` без `ALL`;
- Q11 – `EXCEPT` без `ALL`;

Проверяются результат, названия и порядок колонок, сортировка. Необязательные слова `INNER` и `OUTER` писать не нужно; псевдонимы таблиц и форматирование можно выбирать самостоятельно.

В Q12–Q16 проверяется только результат. Решение можно записать любым корректным способом.

---

### Q01, 1 балл

Соедините `course.airports_master` с `course.departures_feed` через `JOIN` по `airport_code`.

Выведите код аэропорта из справочника как `airport_code`, его название `airport_name` и идентификатор вылета `departure_id`. Отсортируйте по `airport_code`, затем по `departure_id`.

### Q02, 1 балл

Повторите Q01 через `JOIN ... USING (airport_code)`.

Выведите общий `airport_code`, `airport_name`, `departure_id`. Отсортируйте по `airport_code`, затем по `departure_id`.

### Q03, 1 балл

Теперь замените соединение из Q01 на `LEFT JOIN`, чтобы сохранить **все** строки `course.airports_master`, в том числе аэропорт без события вылета.

Выведите `airport_code`, `airport_name`, `departure_id`. Отсортируйте по `airport_code`, затем по `departure_id`.

### Q04, 1 балл

Получите тот же результат, что в Q03, но переставьте таблицы:

- слева – `course.departures_feed`;
- справа – `course.airports_master`;
- соединение – `RIGHT JOIN`.

Выведите `airport_code` и `airport_name` из справочника, затем `departure_id` из выгрузки. Отсортируйте по `airport_code`, затем по `departure_id`.

### Q05, 1 балл

Соедините `course.airports_master` и `course.departures_feed` через `FULL JOIN`.

Выведите:

- код из справочника как `master_airport`;
- `airport_name`;
- код из выгрузки как `feed_airport`;
- `departure_id`.

Сохраните несовпавшие строки обеих сторон. Отсортируйте по `master_airport`, затем по `feed_airport`, затем по `departure_id`.

### Q06, 1 балл

Найдите аэропорты из `course.airports_master`, которым не нашлось ни одного события в `course.departures_feed`.

Используйте left anti join: `LEFT JOIN` по `airport_code`, затем проверку `IS NULL` по колонке правой таблицы. Выведите `airport_code`, `airport_name`. Отсортируйте по `airport_code`.

### Q07, 1 балл

Соберите уникальные коды аэропортов, которые встречаются на любом конце рейсов `timetable` с `flight_id <= 20`.

Первая часть возвращает `departure_airport` как `airport_code`, вторая – `arrival_airport`. В обеих частях используйте условие `flight_id <= 20`. Соедините части через `UNION` без `ALL` и отсортируйте общий результат по `airport_code`.

### Q08, 1 балл

Соберите все комбинации аэропортов из `course.airports_master` и классов из `course.fare_classes`.

Используйте `CROSS JOIN`. Выведите `airport_code`, `fare_conditions`. Отсортируйте по `airport_code`, затем по `fare_conditions`. В результате должно быть 6 строк.

---

### Q09, 1,5 балла

Повторите Q07 через `UNION ALL`, чтобы сохранить все появления кодов на концах первых двадцати рейсов.

Обе части должны возвращать одну колонку `airport_code` и использовать условие `flight_id <= 20`. Отсортируйте общий результат по `airport_code`.

### Q10, 1,5 балла

Для рейсов с `flight_id <= 20` найдите коды, которые встречались и как аэропорт отправления, и как аэропорт прибытия.

Первая часть возвращает `departure_airport` как `airport_code`, вторая – `arrival_airport`. В обеих частях используйте условие `flight_id <= 20`. Используйте `INTERSECT` без `ALL`; отсортируйте результат по `airport_code`.

### Q11, 1,5 балла

Для рейсов с `flight_id <= 20` найдите коды, которые встречались как аэропорт отправления, но не встречались как аэропорт прибытия.

Первая часть возвращает `departure_airport` как `airport_code`, вторая – `arrival_airport`. В обеих частях используйте условие `flight_id <= 20`. Используйте `EXCEPT` без `ALL`; отсортируйте результат по `airport_code`.

---

### Q12, 1,5 балла

Теперь переходим к `bookings`. Для билетов бронирования `0000EG` выведите `ticket_no`, `passenger_name`, `flight_id`, `price`.

Один из простых способов – соединить `tickets` с `segments` через `JOIN` по `ticket_no`. Отсортируйте по `ticket_no`, затем по `flight_id`.

### Q13, 1,5 балла

Для рейсов из `timetable` с `flight_id <= 10` подпишите:

- название аэропорта отправления как `departure_airport_name`;
- модель самолёта как `airplane_model`.

Можно использовать два обычных `JOIN`: с `airports` и `airplanes`. Выведите `flight_id` и две перечисленные колонки. Отсортируйте по `flight_id`.

### Q14, 1,5 балла

Для всех рейсов из `flights` с `flight_id <= 10` найдите дорогие сегменты. В этой задаче **дорогой сегмент** – строка `segments` с ценой `price >= 20000`.

Выведите `flight_id`, `status`, номер билета найденного сегмента `ticket_no` и цену сегмента `price`. Рейсы без дорогих сегментов должны остаться. Один из способов – использовать `LEFT JOIN` с `segments` и разместить условие `s.price >= 20000` в `ON`. Отсортируйте по `flight_id`, затем по `ticket_no`.

### Q15, 1,5 балла

Продолжите Q12: к билетам бронирования `0000EG` добавьте сегменты, а затем строку рейса из `flights`. Это можно записать цепочкой из двух `LEFT JOIN`.

Выведите `ticket_no`, `passenger_name`, `flight_id`, статус рейса как `flight_status`. Отсортируйте по `ticket_no`, затем по `flight_id`.

### Q16, 1,5 балла

Для прибывших рейсов из `flights` с `flight_id <= 10` поставьте друг под другом:

- плановый вылет `scheduled_departure` как `event_time` и `event_type = 'scheduled_departure'`;
- фактический вылет `actual_departure` как `event_time` и `event_type = 'actual_departure'`.

Обе части возвращают `flight_id`, `event_time`, `event_type`. В обеих частях используйте условия `status = 'Arrived'` и `flight_id <= 10`; во второй дополнительно оставьте только строки с известным `actual_departure`. Удобный способ поставить результаты друг под другом – `UNION ALL`. Отсортируйте общий результат по `flight_id`, затем по `event_type`.
