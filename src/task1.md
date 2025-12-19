# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx
           Index Cond: (category IS NULL)
   ```
   
   *Объясните результат:*
   Использовался BRIN-индекс, поэтому оптимизатор построил bitmap index scan и затем bitmap heap scan. Индекс быстро проверил сводку по страницам, но совпадений не оказалось (в наборе нет NULL категорий), поэтому вернулось 0 строк с минимальными затратами.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books
     Recheck Cond: (category = 'INDEX')
     Rows Removed by Index Recheck: 150000
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx
           Index Cond: (category = 'INDEX')
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Значение категории не встречается, а BRIN хранит лишь диапазоны. Из-за низкой селективности индекс вернул много «грязных» страниц (lossy bitmap), после проверки по таблице все строки были отфильтрованы условием по автору; итог — 0 строк и затраты на проверку 1225 блоков.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort
     Sort Key: category
     ->  HashAggregate
           Group Key: category
           ->  Seq Scan on t_books
   ```
   
   *Объясните результат:*
   Для `DISTINCT` оптимизатор сделал последовательное чтение всей таблицы, агрегировал уникальные категории в hash-aggregate (6 значений), затем отсортировал их. Индексы не использовались, потому что нужно просканировать все значения и результат маленький.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate
     ->  Seq Scan on t_books
           Filter: (author LIKE 'S%')
   ```
   
   *Объясните результат:*
   Выбрана последовательная выборка: селективность условия крайне низкая (в итоге 0 строк), дешевле просканировать всю таблицу, чем идти через BRIN по строковым шаблонам. Итоговый COUNT агрегирует результаты сканирования.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate
     ->  Seq Scan on t_books
           Filter: (lower(title) LIKE 'o%')
   ```
   
   *Объясните результат:*
   Несмотря на индекс по `LOWER(title)`, оптимизатор выбрал seq scan: ожидается около 750 совпадений из 150k (средняя селективность), а обход всего массива страниц дешевле, чем чтение через индекс с частыми переходами. Фактически нашёлся 1 элемент, поэтому сканирование оказалось дороже ожидаемого.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books
     Recheck Cond: (category = 'INDEX' AND author = 'SYSTEM')
     Rows Removed by Index Recheck: 8833
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx
           Index Cond: (category = 'INDEX' AND author = 'SYSTEM')
   ```
   
   *Объясните результат:*
   Составной BRIN значительно сузил число страниц (73 против 1225), поэтому bitmap heap scan стал быстрее, хотя совпадений по-прежнему нет. Комбинация ключей даёт более компактные сводки по блокам и снижает «грязность» битмапа.
