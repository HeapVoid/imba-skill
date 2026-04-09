---
name: imba-async
description: Async patterns in Imba — async/await, imba.awaits, Queue (parallel promise manager), await wait Xms, imba.commit! after async ops.
---

# Async in Imba

## Async / Await

`async` не пишется явно — любой метод с `await` автоматически асинхронный.

```imba
def fetchUser id
	let response = await window.fetch("/api/users/{id}")
	await response.json!

tag user-profile
	user = null

	def mount
		user = await fetchUser(userId)
		# imba.commit! not needed — mount is awaited by the tag lifecycle

	<self>
		if user then <p> "Hello {user.name}"
		else <p> "Loading..."
```

**После await — нужен `imba.commit!`** (если не в lifecycle методе):

```imba
def load
	data = await api.fetch!
	imba.commit!

# В lifecycle методах (mount, routed) — commit автоматический
def mount
	data = await api.fetch!   # commit не нужен
```

## await wait — задержка

```imba
await wait 500ms
await wait 2s
```

## imba.awaits — ждать изменения observable

Выполняет callback при каждом изменении observable-свойства до тех пор, пока callback не вернёт truthy:

```imba
let user = new class User
	@observable first_name = 'first'

setTimeout(&, 100ms) do user.first_name = 'updated'
setTimeout(&, 200ms) do user.first_name = 'again'

await imba.awaits do
	console.log user.first_name
	user.first_name is 'updated'
# Завершится когда first_name станет 'updated'
```

## Queue — параллельный менеджер промисов

Встроенный в imba (undocumented). `Set`, который отслеживает выполнение промисов и генерирует события `busy` / `idle`.

```imba
import { Queue } from 'imba'

let q = new Queue

q.on('busy') do console.log 'Есть активные задачи'
q.on('idle') do console.log 'Все завершены'
q.on('add', do(task) console.log 'Добавлена:', task)
q.on('delete', do(task) console.log 'Завершена:', task)

q.add(fetch('/api/images'))
q.add(fetch('/api/tools'))
q.add(do someAsyncWork!)

await q.idle
console.log 'Готово!'
```

**API:**

| Метод / свойство | Описание |
|-----------------|----------|
| `q.add(promise)` | Добавить промис (или функцию → вызовет её) |
| `q.idle` | Promise — резолвится когда очередь пуста |
| `q.idleΦ` | Boolean — true если очередь пуста |
| `q.size` | Количество активных задач |

**Пример: индикатор загрузки**

```imba
import { Queue } from 'imba'

tag image-loader
	queue = new Queue
	loading = false

	def mount
		queue.on('busy') do loading = true, imba.commit!
		queue.on('idle') do loading = false, imba.commit!

	def loadImages urls
		for url in urls
			queue.add(fetch(url).then(do |r| r.blob!))

	def render
		<self>
			if loading then <div> "Загрузка..."
			else <div> "Готово"
```

> Queue используется внутри роутера imba для отслеживания асинхронных навигаций.
