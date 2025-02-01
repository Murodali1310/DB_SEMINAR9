## Индексы. Интеграция с другими ЯП

### 1. Оптимизация

### 1.1. Жизненный цикл запроса, план запроса.

#### 1.1.1. Жизненный цикл запроса.

*Мы написали запрос, что происходит дальше?*

1. Создается **подключение к СУБД**. В СУБД отправляется запрос в виде обычного текста.
2. Парсер **проверяет корректность синтаксиса** запроса и создает **дерево запроса**.
3. Система переписывания запросов преобразует запрос – получаем обновленное дерево запроса; используется [система правил](https://ru.wikipedia.org/wiki/%D0%A1%D0%B5%D0%BC%D0%B0%D0%BD%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B0%D1%8F_%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F_%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2_%D0%A1%D0%A3%D0%91%D0%94#:~:text=%D0%A1%D0%B5%D0%BC%D0%B0%D0%BD%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B0%D1%8F%20%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%A1%D0%A3%D0%91%D0%94%20%E2%80%94%20%D0%BF%D1%80%D0%BE%D1%86%D0%B5%D1%81%D1%81,%D0%BF%D1%80%D0%B8%D0%B3%D0%BE%D0%B4%D0%BD%D1%83%D1%8E%20%D0%B4%D0%BB%D1%8F%20%D0%B4%D0%B0%D0%BB%D1%8C%D0%BD%D0%B5%D0%B9%D1%88%D0%B8%D1%85%20%D1%88%D0%B0%D0%B3%D0%BE%D0%B2%20%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8.])

4. Планировщик / оптимизатор создает **план запроса**.
5. Обработчик рекурсивно обходит план запроса и получает результирующий набор строк.

**Дерево запроса** — специальное внутреннее представление SQL-запросов с полным его разбором по ключевым параметрам:
* Тип команды (`SELECT, UPDATE, DELETE, INSERT`);
* Список используемых отношений;
* Целевое отношение, в которое будет записан результат;
* Список полей (`*` преобразуется в полный список всех полей);
* Список ограничений (которые указаны в `WHERE`);
* и т.д.

#### 1.1.2. План запроса. Как читать, на что обращать внимание. Операторы EXPLAIN и ANALYZE.

**Планировщик (planner)** – компонент PostgreSQL, пытающийся выработать наиболее эффективный способ выполнения запроса SQL.

В плане выполнения содержится информация о том, как будет организован просмотр таблиц, задействованных в запросе, сервером базы данных.

Оператор [EXPLAIN](https://postgrespro.com/docs/postgresql/9.6/sql-explain):

**Индекс** — это объект базы данных, который реализует связь между строками таблицы по определённым ключам, позволяя значительно ускорить поиск и выборку данных. Индексы хранятся отдельно от таблиц, но могут быть воссозданы на основе информации, содержащейся в самих таблицах, поэтому их относят к вспомогательным структурам. Кроме того, индексы применяются для обеспечения некоторых ограничений целостности, например уникальности данных в определённых столбцах. Дальше в пункте 1.2 рассмотрим индексы более подробно.

**Функции:**
* Выводит план выполнения, генерируемый планировщиком PostgreSQL для заданного оператора.
* Показывает, как будут сканироваться таблицы, затрагиваемые оператором — просто последовательно, по индексу и т.д.
* Показывает, какой алгоритм соединения будет выбран для объединения считанных из таблиц строк.
* Показывает ожидаемую *стоимость* (в условных единицах) выполнения запроса.
* ОТСУТСТВУЕТ в стандарте SQL.

**Синтаксис:**
```sql
EXPLAIN [ ( option [, ...] ) ] statement
EXPLAIN [ ANALYZE ] [ VERBOSE ] statement

where option can be one of:
ANALYZE [ boolean ]
VERBOSE [ boolean ]
COSTS [ boolean ]
BUFFERS [ boolean ]
TIMING [ boolean ]
FORMAT { TEXT | XML | JSON | YAML }
```

```sql
    INSERT INTO my_table ...;
    EXPLAIN SELECT * FROM my_table;
- - - -
QUERY PLAN
Seq Scan on my_table (cost=0.00..18334.00 rows=1000000 width=37)
```

**Что это значит?**
* Данные читаются методом Sed Scan (см. следующий вопрос)
* Данные читаются из таблицы my_table
* cost — затраты (в некоторых условных единицах) на получение первой строки..всех строк
* rows — приблизительное количество возвращаемых строк при выполнении операции Seq Scan (никакие строки не вычитываются, значение приблизительное)
* width — средний размер одной строки в байтах

При повторном вызове покажет старую статистику, необходимо вызвать команду ANALYZE для ее обновления.

Оператор [ANALYZE](https://postgrespro.ru/docs/postgresql/9.6/sql-analyze):

**Функции:**
* Собирает статистическую информацию о содержимом таблиц в базе данных и сохраняет результаты в системном каталоге `pg_statistic`;
* Без параметров анализирует все таблицы в текущей базе данных.
* Если в параметрах передано имя таблицы, обрабатывает только заданную таблицу.
* Если в параметрах передан список имен столбцов, то сбор статистики запустится только по этим столбцам.
* ОТСУТСТВУЕТ в стандарте SQL.

**Синтаксис:**
```sql
ANALYZE [ VERBOSE ] [ table_name [ ( column_name [, ...] ) ] ]
```

```sql
ANALYZE my_table;
EXPLAIN SELECT * FROM my_table;

EXPLAIN (ANALYZE) SELECT * FROM my_table;
- - - -
QUERY PLAN
Seq Scan on foo (cost=0.00..18334.10 rows=1000010 width=37)
(actual time=0.402..97.000 rows=1000010 loops=1)

Planning time: 0.042 ms
Execution time: 138.229 ms
```

Теперь запрос будет исполняться реально.
* actual time — реальное время в миллисекундах, затраченное для получения первой строки и всех строк соответственно.
* rows — реальное количество строк, полученных при Seq Scan.
* loops — сколько раз пришлось выполнить операцию Seq Scan.
* Planning time — время, потраченное планировщиком на построение плана запроса.
* Execution time — общее время выполнения запроса.

---

### 1.2. Индексы. Определение, условия использования, способы сканирования.

**Индекс** — это объект базы данных, который реализует связь между строками таблицы по определённым ключам, позволяя значительно ускорить поиск и выборку данных. Индексы хранятся отдельно от таблиц, но могут быть воссозданы на основе информации, содержащейся в самих таблицах, поэтому их относят к вспомогательным структурам. Кроме того, индексы применяются для обеспечения некоторых ограничений целостности, например уникальности данных в определённых столбцах.

**Аналогии с другими языками программирования и структурами данных:**
* Хеш-таблицы (Hash Map, Dictionary) — индексы по типу хеш-индексов похожи на работу с хеш-таблицами, когда поиск элемента занимает постоянное время (O(1)) при равномерном распределении.
* B-деревья, бинарные деревья поиска — индексы вида B-Tree напоминают деревья поиска в языках программирования, где эффективен поиск по диапазонам и сортированным наборам данных.

В PostgreSQL 9.6 встроены *шесть разных видов индексов*.

**Свойства:**
* Все индексы — вторичные, они отделены от таблицы. Вся информация о них содержится в системном каталоге.
* При добавлении/изменении данных, связанных с индексом, индекс каждый раз перестраивается (это замедляет выполнение запроса).
* Внутри могут лежать разные математические структуры (B-дерево, черно-красное дерево…)
* Индексы могут быть многоколоночными (поддержание условия на несколько полей).
* Индексы связывают ключи и TID (tuple id -  #page: #offset) — номер страницы и строки на ней.
* Обновление полей таблицы, по которым не создавались индексы, не приводит к перестроению индексов (Heap-Only Tuples, HOT).

*Оптимизация HOT:* при апдейте строки, если это возможно, Postgres поставит новую копию строки сразу после старой копии строки. Также в старой копии строки проставляется специальная метка, указывающая на то, что новая копия строки находится сразу после старой. Поэтому обновлять все индексы не нужно.

**Условия использования:**
* Совпадают оператор и типы аргументов.
* Индекс валиден.
* Важен порядок полей внутри многоколоночного индекса, чтобы накладывать условия, ожидая, что оптимизатор выберет индекс.
* План с его использованием — оптимален (минимальная стоимость).
* Всю информацию Postgres берет из системного каталога.


**Синтаксис:**
```sql
CREATE [UNIQUE] INDEX [CONCURRENTLY] [name] ON table_name [USING METHOD]...
CREATE INDEX ON my_table(column_2);
ALTER INDEX [IF EXISTS] name RENAME TO new_name
DROP INDEX [CONCURRENTLY] [IF EXISTS] name [, ...] [CASCADE|RESTRICT]

ALTER INDEX [IF EXISTS] name RENAME TO new_name
ALTER INDEX [IF EXISTS] name SET TABLESPACE tablespace_name
ALTER INDEX [IF EXISTS] name SET (storage_parameter = value [, ... ])
ALTER INDEX [IF EXISTS] name RESET (storage_parameter [, ... ])
DROP INDEX [CONCURRENTLY] [IF EXISTS] name [, ...] [CASCADE|RESTRICT]
```
**Задачи, где индексы **нужны****

* 1. Поиск по столбцу с уникальными значениями (например, `ID`)

**Задача:** У вас есть таблица с данными о пользователях, где `ID` является уникальным идентификатором.

**Запрос:** 
```sql
SELECT * FROM users WHERE id = 123;


**Общая информация**
* Индексы работают тем лучше, чем выше селективность условия, то есть чем меньше строк ему удовлетворяет. При увеличении выборки возрастают и накладные расходы на чтение страниц индекса.
* Ситуация усугубляется тем, что последовательное чтение выполняется быстрее, чем чтение страниц «вразнобой». Это особенно верно для жестких дисков, где механическая операция подведения головки к дорожке занимает существенно больше времени, чем само чтение данных; в случае дисков SSD этот эффект менее выражен.

### 1.3. Способы считывания данных

#### 1.3.1. Seq Scan

**Идея:** Последовательное, блок за блоком, чтение данных таблицы.

**Преимущества:** При большом объеме данных для одного значения индексного поля, работает эффективнее индексного сканирования, так как обычно оно работает с большими блоками данных, поэтому за одну операцию доступа потенциально может выбрать большее количество данных, чем индексное сканирование, соответственно, нужно меньше операций доступа, скорость выше.

**Недостатки:** Обычно выполняется гораздо медленнее индексного сканирования, так как считывает все данные таблицы.

#### 1.3.2. Index Scan

**Идея:** Используется индекс для условий `WHERE` (селективность условия), читает таблицу при отборе строк.

**Преимущества:** При селективности условия время N → lnN. (В результате выполнения запроса выбирается значительно меньше строк, чем их кол-во в странице)

**Недостатки:** Если будем собирать индекс по всем полям, он будет весить зачастую значительно больше, чем данные в таблице. При увеличении выборки возрастают шансы, что придется возвращаться к одной и той же табличной странице несколько раз. (В таком случае оптимизатор переключается на Bitmap Scan)

#### 1.3.3. Bitmap Index Scan

**Идея:** Сначала Index Scan, затем контроль выборки по таблице. В большей части работа со строками (индекс по строке, битовая карта страниц, последующий отбор)

**Преимущества:** Эффективно для большого количества строк.

**Недостатки:** Не ускоряет работу, если условие не является селективным. Выборка может оказаться слишком велика для объема оперативной памяти, тогда строится только битовая карта страниц — она занимает меньше места, но при чтении страницы приходится перепроверять условия для каждой хранящейся там строки.

* В случае почти упорядоченных данных построение битовой карты — лишний шаг, обычное индексное сканирование будет таким же.
* Создается битовая карта, где предполагаем, что в соответствии с собранной статистикой наши строки удовлетворяют нашему условию; в ней есть странички;
* Если условия наложены на несколько полей таблицы, и эти поля проиндексированы, сканирование битовой карты позволяет (если оптимизатор сочтет это выгодным) использовать несколько индексов одновременно. Для каждого индекса строятся битовые карты версий строк, которые затем побитово логически умножаются (если выражения соединены условием AND), либо логически складываются (если выражения соединены условием OR).

#### 1.3.4. Index Only Scan

**Идея:** Практически не обращаемся к таблице, все необходимые значения в индексе.

**Преимущества:** Очень быстрая операция.

**Недостатки:** Может применяться только тогда, когда индекс включает все необходимые для выборки поля, и дополнительный доступ к таблице не требуется.

*Если индекс уже содержит все необходимые для запроса данные, то индекс называется покрывающим.*

---

### 2. Интеграция с другими ЯП

### 2.1. Взаимодействие из-под Python

[Python DB API 2.0](https://www.python.org/dev/peps/pep-0249/) - стандарт интерфейсов для пакетов, работающих с БД. "Набор правил", которым подчиняются отдельные модули, реализующие работу с конкретными базами данных.
[Ноутбук с примерами](./examples/examples.ipynb).

### 2.2. Курсоры

**Курсор** – специальный объект, выполняющий запрос и получающий его результат.

Вместо того чтобы сразу выполнять весь запрос, есть возможность настроить **курсор**, инкапсулирующий запрос, и затем получать результат запроса по нескольку строк за раз. Одна из причин так делать заключается в том, чтобы избежать переполнения памяти, когда результат содержит большое количество строк.

В PL/pgSQL:
```sql
name [ [ NO ] SCROLL ] CURSOR [ ( arguments ) ] FOR query;
```

```sql
DECLARE
    curs1 refcursor;
    curs2 CURSOR FOR SELECT * FROM tenk1;
    curs3 CURSOR (key integer) FOR SELECT * FROM tenk1 WHERE unique1 = key;
```


## Практические задачи (Индексы)

   - Создайте таблицу `items(id SERIAL PRIMARY KEY, name TEXT, category_id INT, price NUMERIC)`.
   - Заполните таблицу `items` случайными данными (например, 100000 строк).
   - Выполните запрос:
     ```sql
     EXPLAIN (ANALYZE) SELECT * FROM items WHERE name = 'SomeName';
     ```
     Обратите внимание на используемый метод чтения данных и затраты.
   - Создайте индекс по полю `name`:
     ```sql
     CREATE INDEX idx_items_name ON items(name);
     ```
   - Повторите запрос и сравните план: стало ли быстрее, использовался ли теперь индекс?

   - Выполните:
     ```sql
     EXPLAIN (ANALYZE) SELECT * FROM items WHERE price > 500;
     ```
     Посмотрите, какой метод сканирования выбран.
   - Создайте индекс по полю `price`:
     ```sql
     CREATE INDEX idx_items_price ON items(price);
     ```
   - Повторите запрос и сравните результаты.  
     Изменилось ли время выполнения? Какой метод сканирования теперь используется?

   - Попробуйте выполнить запрос:
     ```sql
     EXPLAIN (ANALYZE) SELECT * FROM items
     WHERE category_id = 10 AND price < 100;
     ```
   - Создайте два индекса:
     ```sql
     CREATE INDEX idx_items_category_id ON items(category_id);
     CREATE INDEX idx_items_price ON items(price);
     ```
   - Повторно выполните запрос, обратите внимание, не использует ли теперь PostgreSQL Bitmap Index Scan с объединением битовых карт?

   - Добавьте в запрос только те поля, которые входят в индекс:
     ```sql
     CREATE INDEX idx_items_name_price ON items(name, price);
     ```
   - Выполните:
     ```sql
     EXPLAIN (ANALYZE) SELECT name, price FROM items WHERE name = 'SomeName';
     ```
   - Проверьте, будет ли применен Index Only Scan (особенно, если поле `name` и `price` полностью покрывают запрос).
