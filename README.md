
# Relational Databases: Concepts and Techniques
## Фінальний проєкт — Реляційні бази даних (MySQL)

---

## 📌 Опис проєкту

Цей проєкт демонструє практичну роботу з реляційними базами даних у **MySQL**:
- створення схеми бази даних
- імпорт даних з CSV-файлу
- нормалізація даних до **третьої нормальної форми (3NF)**
- аналітичні SQL-запити
- робота з датами за допомогою вбудованих SQL-функцій
- створення та використання власних SQL-функцій

Набір даних містить історичну інформацію про випадки інфекційних захворювань у різних країнах за роками.

---

## 📁 Структура репозиторію

```
goit-rdb-fp/
│
├── task_1/                         # створення схеми та імпорт даних
│   ├── create_schema.png
│   ├── import_infectious_cases.png
│   └── count_rows.png
│
├── task_2/                         # нормалізація даних до 3НФ
│   ├── create_entities_table.png
│   ├── fill_entities.png
│   ├── create_infectious_cases_norm.png
│   └── insert_infectious_cases_norm.png
│
├── task_3/                         # аналітичні SQL-запити
│   └── rabies_statistics_top10.png
│
├── task_4/                         # робота з датами
│   └── year_date_difference.png
│
├── task_5/                         # користувацькі SQL-функції
│   ├── years_since_function                # years_since / years_since(...) — різниця в роках
│   └── execute_years_since_function        # запит з використанням функції
│
├── task_bonus/                         # користувацькі SQL-функції       
│   ├── cases_per_period_function.png       # cases_per_period(year_cases, divisor)
│   └── execute_cases_per_period.png        # приклад запуску на даних (rabies per month/quarter/halfyear)
│
└── README.md

```

---

## 🗂 Дані

- Файл-джерело: `infectious_cases.csv`
- Імпорт виконано через **Table Data Import Wizard**
- Кількість імпортованих записів: **7271**
- Схема бази даних: `pandemic`

---

## 🧱 Структура бази даних

```sql
CREATE SCHEMA IF NOT EXISTS pandemic;
USE pandemic;
```

---

## 📥 Початкова таблиця

Імпортовані дані збережено у таблиці:

```sql
pandemic.infectious_cases
```

Особливості таблиці:
- денормалізована структура
- повторювані значення `Entity` та `Code`
- числові показники збережені як `TEXT`
- відсутні значення представлені як порожні рядки (`''`)

---

## 🔄 Нормалізація даних (3NF)

### 1️⃣ Таблиця `entities`

Для усунення дублювання назв країн і кодів створено окрему таблицю:

```sql
CREATE TABLE IF NOT EXISTS entities (
  id INT AUTO_INCREMENT PRIMARY KEY,
  entity VARCHAR(50) NOT NULL,
  code VARCHAR(10) NOT NULL,
  UNIQUE KEY uq_entity_code (entity, code)
);
```

Таблиця наповнена унікальними парами `(Entity, Code)` з початкових даних.

```sql
INSERT INTO entities (entity, code)
SELECT DISTINCT Entity, Code
FROM infectious_cases
WHERE Entity IS NOT NULL AND Code IS NOT NULL;
```

---

### 2️⃣ Нормалізована таблиця випадків захворювань

```sql
CREATE TABLE IF NOT EXISTS infectious_cases_norm (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  entity_id INT NOT NULL,
  year INT NOT NULL,
  number_yaws DECIMAL(18,6) NULL,
  polio_cases INT NULL,
  cases_guinea_worm INT NULL,
  number_rabies DECIMAL(18,6) NULL,
  number_malaria DECIMAL(18,6) NULL,
  number_hiv DECIMAL(18,6) NULL,
  number_tuberculosis DECIMAL(18,6) NULL,
  number_smallpox DECIMAL(18,6) NULL,
  number_cholera_cases DECIMAL(18,6) NULL,
  CONSTRAINT fk_cases_entity
    FOREIGN KEY (entity_id) REFERENCES entities(id),
  UNIQUE KEY uq_entity_year (entity_id, year)
);
```

---

### 🔎 Очищення та приведення типів даних

Під час перенесення даних:
- порожні рядки (`''`) були перетворені у `NULL`
- числові значення безпечно приведені до типу `DECIMAL` за допомогою:

```sql
INSERT INTO infectious_cases_norm
(entity_id, year, number_yaws, polio_cases, cases_guinea_worm, number_rabies,
 number_malaria, number_hiv, number_tuberculosis, number_smallpox, number_cholera_cases)
SELECT
  e.id,
  ic.Year,
  CAST(NULLIF(TRIM(ic.Number_yaws), '') AS DECIMAL(18,6)),
  ic.polio_cases,
  ic.cases_guinea_worm,
  CAST(NULLIF(TRIM(ic.Number_rabies), '') AS DECIMAL(18,6)),
  CAST(NULLIF(TRIM(ic.Number_malaria), '') AS DECIMAL(18,6)),
  CAST(NULLIF(TRIM(ic.Number_hiv), '') AS DECIMAL(18,6)),
  CAST(NULLIF(TRIM(ic.Number_tuberculosis), '') AS DECIMAL(18,6)),
  CAST(NULLIF(TRIM(ic.Number_smallpox), '') AS DECIMAL(18,6)),
  CAST(NULLIF(TRIM(ic.Number_cholera_cases), '') AS DECIMAL(18,6))
FROM infectious_cases ic
JOIN entities e
  ON e.entity = ic.Entity AND e.code = ic.Code;
```

Це забезпечує коректні аналітичні обчислення.

---

## 📊 Аналітичні запити

### 🔹 Перевірка кількості записів

```sql
SELECT COUNT(*) AS total_rows
FROM pandemic.infectious_cases;
```

Результат: **7271 запис**, що відповідає кількості рядків у CSV-файлі.

---

### 🔹 Статистика захворювання на сказ (rabies)

Для кожної унікальної пари `(Entity, Code)` обчислено:
- мінімальне значення
- максимальне значення
- середнє значення

Результати відсортовано за середнім значенням у спадаючому порядку та обмежено до 10 рядків.

```sql
SELECT
  e.entity,
  e.code,
  MIN(icn.number_rabies) AS min_rabies,
  MAX(icn.number_rabies) AS max_rabies,
  AVG(icn.number_rabies) AS avg_rabies
FROM infectious_cases_norm icn
JOIN entities e ON e.id = icn.entity_id
WHERE icn.number_rabies IS NOT NULL
GROUP BY e.entity, e.code
ORDER BY avg_rabies DESC
LIMIT 10;
```

---

## 📅 Робота з датами (вбудовані SQL-функції)

```sql
SELECT
  year,
  MAKEDATE(year, 1) AS date_start,
  CURDATE() AS date_now,
  TIMESTAMPDIFF(YEAR, MAKEDATE(year, 1), CURDATE()) AS diff_years
FROM infectious_cases_norm;
```

Запит:
- створює дату `YYYY-01-01`
- обчислює різницю у **повних роках** між цією датою та поточною датою

---

## 🧩 Власні SQL-функції

### 1️⃣ Функція визначення різниці у роках

```sql
CREATE FUNCTION years_since(input_year INT)
RETURNS INT
NOT DETERMINISTIC
BEGIN
  RETURN TIMESTAMPDIFF(
    YEAR,
    MAKEDATE(input_year, 1),
    CURDATE()
  );
END;
```

#### Використання функції

```sql
SELECT
  MAKEDATE(year, 1) AS record_date,
  CURDATE() AS today,
  years_since(year) AS diff_year
FROM infectious_cases_norm
GROUP BY year
ORDER BY year;
```

---

### 2️⃣ Альтернативна функція: середня кількість випадків за період

Функція обчислює середню кількість випадків захворювання за:
- місяць (`12`)
- квартал (`4`)
- півріччя (`2`)

```sql
CREATE FUNCTION cases_per_period(year_cases DECIMAL(18,6), divisor INT)
RETURNS DECIMAL(18,6)
DETERMINISTIC
BEGIN
  IF year_cases IS NULL OR divisor IS NULL OR divisor = 0 THEN
    RETURN NULL;
  END IF;

  RETURN year_cases / divisor;
END;
```

#### Приклад використання

```sql
SELECT
  e.entity,
  icn.year,
  icn.number_rabies,
  cases_per_period(icn.number_rabies, 12) AS rabies_per_month
FROM infectious_cases_norm icn
JOIN entities e ON e.id = icn.entity_id
WHERE icn.number_rabies IS NOT NULL;
```

---

## ✅ Результати проєкту

- ✔ Створено схему бази даних
- ✔ Дані успішно імпортовано
- ✔ Дані нормалізовано до **3NF**
- ✔ Реалізовано аналітичні SQL-запити
- ✔ Використано вбудовані SQL-функції для роботи з датами
- ✔ Створено та застосовано власні SQL-функції
- ✔ Усі запити виконуються коректно та повертають очікувані результати

---

## 🏁 Висновок

Проєкт демонструє:
- правильне проєктування реляційної БД
- коректну нормалізацію даних
- безпечне очищення та приведення типів
- аналітичне мислення та впевнене володіння SQL
- роботу з користувацькими функціями в MySQL
