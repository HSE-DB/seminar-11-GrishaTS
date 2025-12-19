## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster
      Recheck Cond: (category = 'A')
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx
            Index Cond: (category = 'A')
    ```
    
    *Объясните результат:*
    Индекс по `category` даёт bitmap со всеми совпадающими строками (почти половина таблицы), затем bitmap heap scan читает нужные блоки. Из‑за большого числа совпадений обход затронул ~8k страниц, время ~69 мс.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    Таблица физически переупорядочена по индексу `test_cluster_cat_idx`; кластеризация завершилась успешно (команда `CLUSTER`).

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster
      Recheck Cond: (category = 'A')
      Heap Blocks: exact=4163
      ->  Bitmap Index Scan on test_cluster_cat_idx
            Index Cond: (category = 'A')
    ```
    
    *Объясните результат:*
    После кластеризации строки одной категории легли плотнее. Bitmap heap scan читает те же строки, но охватывает вдвое меньше блоков (4163), что сокращает время до ~64 мс.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    До кластеризации: ~69 мс, 8334 блоков. После: ~64 мс, 4163 блока. Упорядочивание сократило количество обращений к диску/буферу примерно вдвое, что дало умеренный прирост производительности на широком выборке (почти половина таблицы).
