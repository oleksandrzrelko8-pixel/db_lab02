# Лабораторна робота 2. Створення складних SQL запитів

## Загальна інформація

- **Здобувач освіти:** Зрелко Олександр Вадимович
- **Група:** [Вкажіть вашу групу]
- **Обраний рівень складності:** 2 (Достатній рівень — "добре")

---

## Виконання завдань

### Рівень 1

#### 1. З'єднання таблиць

**Завдання 1.1: INNER JOIN - список товарів з категоріями та постачальниками**

```sql
SELECT p.product_name, c.category_name, s.company_name, p.unit_price
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
INNER JOIN suppliers s ON p.supplier_id = s.supplier_id
ORDER BY c.category_name, p.product_name;
```

**Результат виконання:**
![INNER JOIN](screenshots/01_inner_join.png.jpg)

**Пояснення:** Запит поєднує дані з трьох пов'язаних таблиць: каталогу товарів (`products`), категорій (`categories`) та постачальників (`suppliers`). Використано з'єднання `INNER JOIN`, тому до результату потрапляють лише ті записи, для яких знайдено точні збіги за зовнішніми ключами (`category_id` та `supplier_id`). Товари без прив'язаної категорії чи постачальника автоматично виключаються з вибірки.

---

**Завдання 1.2: LEFT JOIN - клієнти з кількістю замовлень**

```sql
SELECT c.contact_name, c.customer_type, r.region_name,
       COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN regions r ON c.region_id = r.region_id
GROUP BY c.customer_id, c.contact_name, c.customer_type, r.region_name
ORDER BY order_count DESC;
```

**Результат виконання:**
![LEFT JOIN](screenshots/02_left_join.png.jpg)

**Пояснення:** На відміну від `INNER JOIN`, який відсіює записи без пари, `LEFT JOIN` зберігає абсолютно всі записи з лівої таблиці (`customers`), навіть якщо у клієнта немає жодного замовлення в таблиці `orders`. У таких випадках поле `o.order_id` має значення `NULL`, а агрегатна функція `COUNT(o.order_id)` повертає 0. Це дозволяє проаналізувати всю клієнтську базу разом з неактивними покупцями.

---

**Завдання 1.3: Множинне з'єднання - детальна інформація про замовлення**

```sql
SELECT 
    o.order_id,
    o.order_date,
    c.contact_name AS customer_name,
    p.product_name,
    oi.quantity,
    oi.unit_price,
    ROUND(oi.quantity * oi.unit_price * (1 - oi.discount), 2) AS item_total,
    e.first_name || ' ' || e.last_name AS manager_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN employees e ON o.employee_id = e.employee_id
ORDER BY o.order_id, p.product_name;
```

**Результат виконання:**
![Множинне з'єднання](screenshots/03_multi_join.png.jpg)

**Аналіз складності:** Запит виконує послідовне з'єднання 5 таблиць: відправною точкою є замовлення `orders` ($O$), далі підтягується клієнт `customers` ($C$), позиції чека `order_items` ($I$), найменування товару `products` ($P$) та менеджер `employees` ($E$). Оскільки всі з'єднання виконуються за первинними та зовнішніми B-Tree індексами, PostgreSQL застосовує алгоритм `Hash Join`. Алгоритмічна складність запиту є лінійною від кількості куплених одиниць: $O(I)$, з подальшим сортуванням результату $O(I \log I)$.

---

#### 2. Агрегатні функції

**Завдання 2.1: Статистика товарів за категоріями**

```sql
SELECT c.category_name,
       COUNT(p.product_id) as product_count,
       ROUND(AVG(p.unit_price), 2) as avg_price,
       MIN(p.unit_price) as min_price,
       MAX(p.unit_price) as max_price
FROM categories c
LEFT JOIN products p ON c.category_id = p.category_id
GROUP BY c.category_id, c.category_name
ORDER BY product_count DESC;
```

**Результат виконання:**
![Статистика за категоріями](screenshots/04_aggregates_category.png.jpg)

---

**Завдання 2.2: Продажі за регіонами з використанням HAVING**

```sql
SELECT 
    r.region_name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    ROUND(SUM(oi.quantity * oi.unit_price * (1 - oi.discount)), 2) AS total_sales
FROM regions r
JOIN customers c ON r.region_id = c.region_id
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered'
GROUP BY r.region_id, r.region_name
HAVING SUM(oi.quantity * oi.unit_price * (1 - oi.discount)) > 50000
ORDER BY total_sales DESC;
```

**Результат виконання:**
![Продажі за регіонами](screenshots/05_sales_by_region.png.jpg)

---

**Завдання 2.3: Постачальники з кількістю товарів більше 2**

```sql
SELECT 
    s.company_name,
    s.city,
    COUNT(p.product_id) AS products_supplied
FROM suppliers s
JOIN products p ON s.supplier_id = p.supplier_id
GROUP BY s.supplier_id, s.company_name, s.city
HAVING COUNT(p.product_id) > 2
ORDER BY products_supplied DESC;
```

**Результат виконання:**
![Постачальники HAVING](screenshots/06_suppliers_having.png.jpg)

---

#### 3. Базові підзапити

**Завдання 3.1: Товари з ціною вище середньої по категорії**

```sql
SELECT p.product_name, p.unit_price, c.category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE p.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p.category_id
)
ORDER BY c.category_name, p.unit_price DESC;
```

**Результат виконання:**
![Підзапит у WHERE](screenshots/07_subquery_where.png.jpg)

---

**Завдання 3.2: Клієнти з замовленнями у 2024 році**

```sql
SELECT customer_id, contact_name, city, phone
FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2024
)
ORDER BY contact_name;
```

**Результат виконання:**
![Підзапит з IN](screenshots/08_subquery_in.png.jpg)

---

**Завдання 3.3: Товари з загальною кількістю продажів**

```sql
SELECT 
    p.product_id,
    p.product_name,
    p.unit_price,
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM order_items oi
        WHERE oi.product_id = p.product_id
    ), 0) AS total_units_sold
FROM products p
ORDER BY total_units_sold DESC;
```

**Результат виконання:**
![Підзапит у SELECT](screenshots/09_subquery_select.png.jpg)

---

### Рівень 2

#### 4. Складні з'єднання

**Завдання 4.1: RIGHT JOIN - аналіз категорій та товарів**

```sql
SELECT c.category_name,
       COUNT(p.product_id) as products_count,
       COALESCE(ROUND(AVG(p.unit_price), 2), 0) as avg_price
FROM products p
RIGHT JOIN categories c ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name
ORDER BY products_count DESC;
```

**Результат виконання:**
![RIGHT JOIN](screenshots/10_right_join.png.jpg)

---

**Завдання 4.2: Self-join - співробітники та керівники**

```sql
SELECT e1.first_name || ' ' || e1.last_name as employee,
       e1.title as employee_title,
       COALESCE(e2.first_name || ' ' || e2.last_name, 'Немає (Топ-менеджер)') as manager,
       COALESCE(e2.title, 'Дирекція') as manager_title
FROM employees e1
LEFT JOIN employees e2 ON e1.reports_to = e2.employee_id
ORDER BY e2.last_name NULLS FIRST, e1.last_name;
```

**Результат виконання:**
![Self-join](screenshots/11_self_join.png.jpg)

---

#### 5. Віконні функції

**Завдання 5.1: Ранжування товарів за ціною в категоріях**

```sql
SELECT p.product_name,
       c.category_name,
       p.unit_price,
       RANK() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) as price_rank,
       DENSE_RANK() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) as price_dense_rank,
       ROW_NUMBER() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) as row_num
FROM products p
JOIN categories c ON p.category_id = c.category_id
ORDER BY c.category_name, p.unit_price DESC;
```

**Результат виконання:**
![Віконне ранжування](screenshots/12_window_ranking.png.jpg)

---

**Завдання 5.2: Порівняння замовлень з попередніми датами**

```sql
SELECT 
    o.customer_id,
    c.contact_name,
    o.order_id,
    o.order_date,
    o.freight,
    LAG(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS prev_order_date,
    (o.order_date - LAG(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date)) AS days_since_last_order,
    LEAD(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS next_order_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.customer_id, o.order_date;
```

**Результат виконання:**
![Віконні функції LAG та LEAD](screenshots/13_window_lag_lead.png.jpg)

---

## Висновки

- **Самооцінка:** 4 (добре)
- **Обґрунтування:** Повністю виконано всі завдання Рівня 1 та Рівня 2 відповідно до методичних вимог. Усі 13 SQL-запитів протестовано на навчальній базі даних `technomart` у хмарній СУБД PostgreSQL на платформі Supabase. До звіту додано скріншоти з результатами виконання кожного запиту, надано пояснення бізнес-логіки та різниці між типами з'єднань, а також проведено аналіз алгоритмічної складності множинного з'єднання.

