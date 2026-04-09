---
name: imba-storage
description: imba.locals and imba.session — undocumented proxy wrappers over localStorage/sessionStorage with auto JSON serialization, namespaces, and imba.commit on write.
---

# Storage: imba.locals / imba.session

Встроенные прокси-обёртки над `localStorage` и `sessionStorage` (undocumented).

- Автоматическая сериализация / десериализация через JSON
- Встроенное кэширование — повторные чтения не парсят JSON
- При записи вызывает `imba.commit()` — компоненты ре-рендерятся
- Запись `null` удаляет ключ (`removeItem`)

```imba
import { locals, session } from 'imba'

locals.theme = 'dark'
console.log locals.theme        # 'dark'

locals.user = { name: 'Ivan' }
console.log locals.user.name    # 'Ivan'

locals.theme = null             # удаляет ключ

session.tmp = { step: 1 }
```

## Namespace

```imba
# Все ключи получают префикс "myApp:"
let store = locals('myApp')
store.counter = 42              # сохраняется как "myApp:counter"

# Вложенные namespace
let userstore = locals('myApp')('user')
userstore.name = 'Ivan'         # сохраняется как "myApp:user:name"
```

> Корневые ключи хранятся как `":key"` (пустой namespace-префикс).

## Рекомендуемый паттерн (SSR / private mode)

localStorage может быть недоступен — оборачивай в try/catch:

```imba
import { locals } from 'imba'

export def saveTools tools
	if typeof window == 'undefined'
		return
	try
		locals.favorite_tools = tools
	catch e
		console.warn "[STORAGE] Error:", e

export def loadTools
	if typeof window == 'undefined'
		return []
	try
		return locals.favorite_tools or []
	catch e
		console.warn "[STORAGE] Error:", e
		return []
```
