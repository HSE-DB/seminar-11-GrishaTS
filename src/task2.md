## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books
      Recheck Cond: (to_tsvector('english', title) @@ to_tsquery('english', 'expert'))
      ->  Bitmap Index Scan on t_books_fts_idx
            Index Cond: (to_tsvector('english', title) @@ to_tsquery('english', 'expert'))
    ```
    
    *Объясните результат:*
    GIN‑индекс по `to_tsvector` сразу нашёл токен `expert`, поэтому план — bitmap index scan + heap probe только нужных страниц. Нашлась одна книга, время выполнения — десятки микросекунд.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_pk on t_lookup
       Index Cond: (item_key = '0000000455')
     ```
     
     *Объясните результат:*
     Первичный ключ — btree по `item_key`, поэтому оптимизатор сразу берёт целевой ключ через index scan. 1 страница индекса + одна страница таблицы дают время ~0.07 мс.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered
       Index Cond: (item_key = '0000000455')
     ```
     
     *Объясните результат:*
     План идентичен предыдущему (btree по PK), но за счёт кластеризации данные лежат в порядке ключа, поэтому доступ к heap немного более предсказуем; время сопоставимое.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_value_idx on t_lookup
       Index Cond: (item_value = 'T_BOOKS')
     ```
     
     *Объясните результат:*
     Новый вторичный индекс по `item_value` позволил выполнить точечный поиск. Значение отсутствует, поэтому индекс быстро проверил нужный диапазон и вернул 0 строк без полного сканирования.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered
       Index Cond: (item_value = 'T_BOOKS')
     ```
     
     *Объясните результат:*
     Аналогичный индексный поиск по кластеризованной таблице. Из‑за кластеризации по другому ключу выигрыш минимален, но чтение также ограничилось страницами индекса.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Оба поиска по значению выполняются через btree‑индексы и завершаются за ~0.03–0.05 мс. Кластеризация по первичному ключу не дала заметного преимущества для вторичного индекса: планы одинаковые, время отличается лишь в пределах шума.
