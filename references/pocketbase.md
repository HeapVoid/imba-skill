# PocketBase Hooks in Imba Reference

Backend hooks for PocketBase written in Imba (chain: Imba → JS → Goja runtime).

**Документация:**
- Context7: `mcp_io_github_ups_get-library-docs` с ID `/websites/pocketbase_io_jsvm`
- Сайт: https://pocketbase.io/docs/ и https://pocketbase.io/jsvm/

---

## Критичные правила Goja

### File scope в `.pb.js` — callback-ы `routerAdd` изолированы

PocketBase 0.36+ выполняет каждый `routerAdd` callback в **изолированном Goja-контексте**. Переменные и функции, объявленные на уровне файла в `.pb.imba` (хук-файлах), **не видны** внутри callback-ов `routerAdd`.

Это касается **только файлов с хуками** (`.pb.imba` → `.pb.js`). В обычных модулях (например, `utils.imba` → `utils.js`) функции на уровне файла работают нормально, видят друг друга и корректно экспортируются через `module.exports`.

**Решение:** выносить shared-хелперы в `utils.imba` (или другой модуль), экспортировать через `module.exports`, и импортировать через `require()` внутри каждого callback-а.

```imba
# ✅ utils.imba — обычный модуль, file-scope работает
const isProjectMember = do(projectId, userId)
	# ... может вызывать другие функции из этого же файла
	return rows.length > 0

module.exports = { isProjectMember }

# ✅ api.chats.pb.imba — хелперы через require внутри callback
routerAdd "GET", "/projects/{projectId}/chats", do(e)
	const { auth, isProjectMember } = require("{__hooks}/utils.js")
	const { request } = auth(e)
	# ...

# ❌ api.chats.pb.imba — file-scope НЕ видна внутри routerAdd
const isProjectMember = do(projectId, userId)
	# ...
routerAdd "GET", "/projects/{projectId}/chats", do(e)
	isProjectMember(projectId, userId) # ReferenceError: isProjectMember is not defined
```

### `require` — только внутри функций

```imba
# ✅
routerAdd "POST", "/api/foo", do(e)
	const { auth } = require('./utils.js')

# ❌ не работает в Goja — на верхнем уровне
const { auth } = require('./utils.js')
```

### Хуки с коллекцией — правильный отступ

Хуки вида `onRecordEnrich`, `onRecordCreateRequest` и т.п. принимают **два аргумента**: коллбек и название коллекции. Оба аргумента — с **одним табом** отступа от имени хука. **Тело коллбека** — с **двумя табами**. `e.next()` — часть тела коллбека, значит тоже **два таба**.

```imba
# ✅ Правильная структура
onRecordEnrich 
	do(e)          # 1 таб — первый аргумент (коллбек)
		# ... тело хука, 2 таба
		e.next()    # 2 таба — внутри коллбека, ОБЯЗАТЕЛЬНО вызвать
	"users"        # 1 таб — второй аргумент (имя коллекции)

# ❌ НЕПРАВИЛЬНО — e.next() на одном уровне с do(e), "users" потерялся
onRecordEnrich
	do(e)
		# ... тело
	e.next()   # ← 1 таб: вне do(e), сломана логика
	"users"    # ← 1 таб: интерпретируется как отдельное выражение

# ❌ НЕПРАВИЛЬНО — "users" внутри do(e)
onRecordEnrich
	do(e)
		# ... тело
		e.next()   # ← 2 таба: ok
		"users"    # ← 2 таба: стало частью тела do(e), не фильтр коллекции
```

> Это касается всех хуков с фильтром по коллекции: `onRecordCreateRequest`, `onRecordUpdateRequest`, `onRecordDeleteRequest`, `onRecordEnrich`, `onRecordAuthRequest` и т.д.

### Транзакции — всегда через `xapp`, не `$app`

```imba
# ✅
$app.runInTransaction do(xapp)
	const user = xapp.findRecordById('users', id)
	user.set('balance', user.get('balance') - 100)
	xapp.save(user)
	return

# ❌ дедлок — $app внутри транзакции
$app.runInTransaction do(xapp)
	$app.save(user)
```

### Циклы — через методы массива, не Imba-синтаксис

```imba
# ✅
collection.forEach do(item) item.process!

# ❌ сложные Imba-циклы могут не работать в Goja
for item in collection
	item.process!
```

---

## Доступ к полям записи

| Поле | Способ |
|------|--------|
| `id` | `record.id` — прямое JS-свойство, всегда |
| `verified`, `email`, `tokenKey` | `record.verified!` — Go-аксессор, только auth-поля |
| `created`, `updated` | `record.get('created')` — системные поля, **не** методы `created()`/`updated()` (они есть только у `ExternalAuth`/`AuthOrigin`, не у `Record`) |
| Любое кастомное поле | `record.get("fieldName")` |

```imba
# ✅
user.id                    # id — прямое свойство
user.verified!             # Go-аксессор для auth-полей
user.get('created')        # системное поле created
user.get('updated')        # системное поле updated
user.get("nonce")          # кастомное поле
request.auth.id            # id авторизованного юзера

# ❌
user.nonce!                # undefined — нет такого Go-метода
user.verified              # undefined — это метод, нужен !
user.created()             # undefined — нет такого метода у Record
user.updated()             # undefined — нет такого метода у Record
```

---

## Поиск записей

```imba
# findRecordById — бросает 404 если не найдена, оборачивай в try
try
	const record = $app.findRecordById('users', id)

# findRecordsByIds — возвращает null если не найдена, безопаснее
const record = $app.findRecordsByIds('users', [id])[0]

# фильтр
$app.findRecordsByFilter("users", "status='active'", "-joined", 100, 0)

# все записи
$app.findAllRecords("item_sets")
```

---

## JSON-поля

Go возвращает объект с артефактами, а не чистый JS-объект:

```imba
const raw = record.get('config')
# ❌ raw = { value: '{"x":1}', marshalJSON, scan, ... }

# ✅ всегда парси через extract из utils.js
const { extract } = require('./utils.js')
const config = extract(record, 'config', {})
```

---

## `set` vs `setRaw` vs `setIfFieldExists`

| Метод | Что делает |
|---|---|
| `record.set(key, value)` | Нормализует значение через field setter коллекции, если поле существует. Для неизвестных полей (custom data) — просто сохраняет. |
| `record.setRaw(key, value)` | Сохраняет **без нормализации** — обходит все field setters. Значение записывается как есть. |
| `record.setIfFieldExists(key, value)` | Сохраняет (с нормализацией) **только если** ключ совпадает с полем коллекции. Для неизвестных ключей — ничего не делает. |

```imba
# set — нормализует через fieldsetter (например, число приводится к int, строка к email-формату и т.п.)
record.set("count", "42")        # сохранится как число 42 если поле number
record.set("meta", { x: 1 })    # JSON-поле сериализуется через PocketBase

# setRaw — без нормализации, значение сохраняется как сырой JS-объект
record.setRaw("balance", 14745)  # полезно для кастомных полей ответа клиенту
record.setRaw("rollup", {...})   # служебные поля, не нужно в БД — только в ответе

# ВАЖНО: поля, выставленные через setRaw, попадают в JSON-ответ
# только если включён withCustomData:
record.withCustomData(true)
```

**Когда что использовать:**
- `set` — для сохранения в БД (поля коллекции)
- `setRaw` + `withCustomData(true)` — для добавления вычисленных/вспомогательных полей в ответ клиенту (не сохраняются в БД)

---

## Операции с данными

```imba
# Создание
const record = new Record($app.findCollectionByNameOrId("users"))
record.set("name", "John")
$app.save(record)

# Обновление
record.set("email", "new@example.com")
$app.save(record)

# Удаление
$app.delete(record)

# Логирование
$app.logger!.info("msg", "key", { extra: "data" })
```

---

## Error Classes

```imba
throw new BadRequestError('...')    # HTTP 400
throw new UnauthorizedError('...')  # HTTP 401
throw new ForbiddenError('...')     # HTTP 403
throw new NotFoundError('...')      # HTTP 404
```

---

## Роуты и хуки

```imba
routerAdd "GET", "/api/foo", do(e)
	const request = e.requestInfo!
	const query = request.method == "GET" ? request.query : request.body
	return e.json(200, { ok: true })
```

### HTTP-заголовки в `requestInfo()`

PocketBase (Goja) **заменяет дефисы на подчёркивания** в ключах заголовков и приводит всё к **нижнему регистру**:

| Отправлен заголовок | Ключ в `request.headers` |
|---------------------|--------------------------|
| `x-agent-token` | `x_agent_token` |
| `Content-Type` | `content_type` |
| `X-Custom-Header` | `x_custom_header` |

```imba
routerAdd "POST", "/my-webhook", do(e)
	const request = e.requestInfo!
	const token = request.headers['x_agent_token']   # ✅ подчёркивания, lowercase
	# const token = request.headers['x-agent-token'] # ❌ НЕ работает — дефисы
	# const token = request.headers['X-Agent-Token'] # ❌ НЕ работает — Go canonical
```

**Запомни:** `request.headers` — это НЕ стандартный HTTP map. PocketBase нормализует ключи перед передачей в Goja.

### JSON-поля (`record.get()` и `JSON.stringify`)

JSON-поля в PocketBase храня��ся как `types.JsonRaw` (Go byte slice). При передаче в Goja `record.get("jsonField")` возвращает **byte slice**, а не JS-объект. Когда ты кладёшь его в объект и вызываешь `JSON.stringify()`, получается **массив байтов** `[123, 34, ...]` вместо объекта.

```imba
# ❌ НЕПРАВИЛЬНО — config будет [123, 34, 99, ...] (byte array)
body: JSON.stringify({
	config: agent.get("config") or {}
})

# ✅ ПРАВИЛЬНО — парсить через String() + JSON.parse()
let agentConfig = {}
const rawCfg = agent.get("config")
if rawCfg
	try agentConfig = JSON.parse(String(rawCfg))
body: JSON.stringify({
	config: agentConfig
})
```

**Правило:** всегда оборачивай `record.get("jsonField")` в `JSON.parse(String(...))` перед вставкой в другой JSON-объект.

```imba

onRecordCreateRequest do(e)
	e.next!
, "users"

onRecordUpdateRequest do(e)
	e.next!
, "users"

cronAdd "Cleanup", "0 2 * * *", do
	# каждый день в 2:00
	return true
```

---

## utils.js — утилиты проекта

### `auth` — авторизация + параметры запроса

```imba
const auth = do(e, required = [])
	const request = e.requestInfo!
	throw new UnauthorizedError("The action requires authentication") if !request.auth
	const query = request.method == "GET" ? request.query : request.body
	required.forEach do(field)
		throw new BadRequestError("Missing '{field}' parameter") if !query[field]
	return { request, query }

# использование
const { request, query } = auth(e, ['userId', 'amount'])
# request.auth → запись авторизованного юзера
```

### `extract` — безопасный парсинг JSON-поля записи

```imba
const extract = do(record, key, fallback = {})
	return fallback if !record or !key or !(record.get isa 'function')
	try return JSON.parse(record.get(key))
	return fallback

# использование — null/undefined или ошибка parse → fallback
const config = extract(record, 'settings', {})
```

---

## RecordUpsertForm — универсальная форма записи

`RecordUpsertForm` — это встроенный класс PocketBase для загрузки, валидации и сохранения данных в запись. Аналог формы из админки — но вызывается программно из JS-хуков.

### Когда использовать

- Создание/обновление записи через REST-эндпоинт (клиент присылает JSON)
- Нужно, чтобы PocketBase сам провалидировал все поля (required, min/max, email-формат и т.д.)
- Не нужно вручную вызывать `record.set()` для каждого поля

### Как работает

```imba
const form = new RecordUpsertForm($app, record)
form.load(query)    # загружает данные из объекта (query params или body)
form.submit!        # валидация + сохранение (всё в одной операции)
```

### Ключевые методы

| Метод | Описание |
|---|---|
| `load(data)` | Загружает данные из объекта в форму и связанную запись. Поля должны совпадать с именами полей коллекции. |
| `submit()` | Запускает валидацию и сохраняет запись. Бросает ValidationError при ошибке. |
| `grantManagerAccess()` | Разрешает изменять системные поля (для auth-коллекций). |
| `grantSuperuserAccess()` | Разрешает изменять **все** поля, включая скрытые. |

### Пример: обновление профиля пользователя

```imba
routerAdd "POST", "/api/users/profile", do(e)
	const { auth } = require("{__hooks}/utils.js")
	const { request, query } = auth(e)

	const form = new RecordUpsertForm($app, request.auth)
	form.load(query)
	form.submit!

	return e.json(200, { ok: true })
```

### Пример: создание записи

```imba
routerAdd "POST", "/api/articles", do(e)
	const { auth } = require("{__hooks}/utils.js")
	const { request, query } = auth(e, ['title', 'content'])

	const collection = $app.findCollectionByNameOrId("articles")
	const record = new Record(collection)
	const form = new RecordUpsertForm($app, record)
	form.load(query)
	form.submit!

	return e.json(201, { id: record.id })
```

### Важные нюансы

1. **`load()` НЕ сохраняет** — только заполняет форму. Сохранение только через `submit()`.
2. **`submit()` делает и валидацию, и сохранение** — не нужно вызывать `validate()` отдельно (такого метода нет).
3. **Поля должны совпадать** с именами в коллекции. Кастомные данные (не поля коллекции) игнорируются.
4. **Для auth-коллекций** (`users`) по умолчанию нельзя менять системные поля (`email`, `password`, `verified`). Используй `form.grantManagerAccess!` или `form.grantSuperuserAccess!`.
5. **Работает в транзакциях** — если нужно несколько операций атомарно:
   ```imba
   $app.runInTransaction do(xapp)
   	const form = new RecordUpsertForm(xapp, record)
   	form.load(query)
   	form.submit!
   	# другие операции с xapp...
   	return
   ```
