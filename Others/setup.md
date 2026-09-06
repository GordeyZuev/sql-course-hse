# Подключение к учебной базе

`PostgreSQL` работает в `Yandex Cloud`. Сервер ставить не нужно. Нужен выбрать способ подключения, наиболее удобный вам. Выбор большой:

- `pgAdmin` [4](https://www.pgadmin.org/download/) – графический клиент от разработчиков PostgreSQL;
- `DBeaver` [Community](https://dbeaver.io/download/) – графический клиент, удобен, если вы уже работаете с несколькими базами;
- `psql` из [пакета PostgreSQL](https://www.postgresql.org/download/) – терминал.

Параметры подключения (для студентов):


| Поле     | Значение                                        |
| -------- | ----------------------------------------------- |
| Host     | `c-c9qioj3cuqplfifu9iof.ro.mdb.yandexcloud.net` |
| Port     | `6432`                                          |
| Database | `demo`                                          |
| User     | `student`                                       |
| Password | Будет в чате                                    |
| SSL mode | `require`                                       |


Пароль преподаватель передаёт закрытым каналом.

## Проверка, что вы внутри учебной базы

В любом клиенте достаточно одного запроса:

```sql
SELECT bookings.version();
```

Ожидается ровно:

```text
PostgresPro 2025-09-01 (91 days)
```

Если версия другая или функции нет – вы не в той базе.

## Вариант A. `pgAdmin`

1. Скачайте `pgAdmin` [4](https://www.pgadmin.org/download/) под вашу ОС и установите.
2. Правый клик по **Servers** → **Register → Server…**.
3. **General**: имя, например `HSE SQL 2026`.
4. **Connection**: host, port, database `demo`, user `student`, пароль из закрытого канала.
5. **SSL**: режим `require`. Root certificate оставьте пустым.
6. **Save**, затем `demo → Tools → Query Tool`.
7. Выполните `SELECT bookings.version();`.

Документация клиента: [pgAdmin](https://www.pgadmin.org/docs/).

## Вариант B. `DBeaver`

1. Скачайте `DBeaver` [Community](https://dbeaver.io/download/) (для курса хватает бесплатной Community, не PRO).
2. **Database → New Database Connection** → **PostgreSQL**. Если драйвер предложит скачаться – согласитесь.
3. Вкладка **Main**: Host, Port `6432`, Database `demo`, Username `student`, пароль.
4. Вкладка **SSL**: включите SSL, режим `require`.
5. **Test Connection** → **Finish**.
6. Правый клик по подключению → **SQL Editor → New SQL Script**.
7. Выполните `SELECT bookings.version();`.

Слева дерево схем (`bookings`), справа редактор SQL. Выделяйте один запрос перед запуском, иначе `DBeaver` может выполнить весь файл. Документация: [создание подключения](https://github.com/dbeaver/dbeaver/wiki/Create-Connection). Порт по умолчанию в клиенте – `5432`, у нас облако слушает `6432`.

## Вариант C. `psql` из терминала

Проверка, что клиент установлен:

```bash
psql --version
```

Подключение:

```bash
export PGSSLMODE=require
psql -h c-c9qioj3cuqplfifu9iof.ro.mdb.yandexcloud.net -p 6432 -U student -d demo
```

`-h` – host, `-p` – port, `-U` – user, `-d` – database.

После `Password for user student:` введите пароль. Символы не отображаются – так и должно быть. Успех: приглашение `demo=>`.

```sql
SELECT bookings.version();
```

## Терминальные команды `psql`: `\x`, `\d`, `\o`, `\q`

Это не SQL. Это команды самого клиента. Точка с запятой им не нужна. Полный список: `\?` или [psql в Postgres Pro](https://postgrespro.ru/docs/postgresql/current/app-psql).


| Команда          | Что делает                                                                  |
| ---------------- | --------------------------------------------------------------------------- |
| `\d`             | список таблиц и view в текущем `search_path`                                |
| `\d airports`    | колонки, типы и ключи объекта `airports`                                    |
| `\dt bookings.*` | таблицы схемы `bookings`                                                    |
| `\dn`            | список схем                                                                 |
| `\x`             | «широкая» строка: каждая колонка с новой строки. Повторный `\x` выключает   |
| `\o result.txt`  | писать результаты следующих запросов в файл. `\o` без имени – снова в экран |
| `\q`             | выход из `psql`                                                             |


Пример:

```text
demo=> \x
Expanded display is on.
demo=> SELECT airport_code, airport_name, city FROM airports LIMIT 1;
demo=> \x
demo=> \d airports
demo=> \q
```

Если приглашение стало `demo->`, вы не закрыли statement или кавычки. Допишите `;` или прервите ввод: `Ctrl+C`.
