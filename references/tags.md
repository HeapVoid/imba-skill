# Imba Tags Reference

Tags, components, lifecycle, slots, refs, router, imba.mount, custom elements, tag inheritance.

> **Forms and `bind=`:** see `references/data-binding.md`
> **Events and modifiers:** see `references/events.md`

---

## Tags (Components)

### Definition
```imba
# Global component (kebab-case, auto-available everywhere after import)
tag my-card
	<self>
		<div.card>
			<slot>

# Local component (PascalCase, needs explicit import wherever used)
tag Box
	<self> ...
```

**Import strategy:** Import all components in `index.imba` — kebab-case tags become globally available. PascalCase tags require per-file imports.

### Properties (Reactive by Default)
```imba
tag user-badge
	name = "Anonymous"    # Just assign — reactive!
	role = "user"

	<self>
		<span> "{name} ({role})"

# Usage: <user-badge name="Alice" role="admin">
```

### Getters & Methods
```imba
tag user-card
	first = ''
	last = ''

	get name do "{first} {last}"

	def update data
		first = data.first
		last = data.last

	<self>
		<span> name
```

### Named Elements (`$name`)

**Syntax: `$name` directly on element tag, no space before `$`**

```imba
tag app-panel
	<self>
		<button @click=($input.value += 'Hi')> "Write"
		<input$input type='text'>

# ❌ WRONG - $name on class
<.wear$wear>

# ❌ WRONG - space before $
<path $wfill d="...">

# ✅ CORRECT
<$wear .wear>
<div$wear .wear>
<path$wfill d="...">
```

**Real HTML `id` attribute + reference — use `#name`:**
```imba
<#wear .wear>      # Sets id="wear" and creates $wear reference
<div#wear .wear>
```

### Events

See `references/events.md` for full event modifier system, custom modifiers, hotkeys, error boundaries, global event bus, and `def on$name` raw handler control.

### Conditional & Dynamic Rendering
```imba
<self>
	if loggedIn
		<dashboard-view>
	else
		<login-form>

	for item in items
		<item-card data=item>

	# Dynamic tag
	<{popup}> if popup
```

### `key=` — List Rendering

Without `key=`, Imba doesn't know which node maps to which item when list changes.

```imba
for item in items
	<item-card key=item.id data=item>

# With ease animation — key= required
for item in items
	<.card key=item.id ease>
		<span> item.name
```

| Ситуация | `key=` нужен? |
|----------|--------------|
| Статичный список | Нет |
| Добавление / удаление | Да |
| Переупорядочивание | Обязательно |
| Элементы с внутренним состоянием | Да |
| Список с `ease` | Да |

### Lifecycle
```imba
tag my-component
	def mount
		# Added to DOM

	def unmount
		# Being removed

	def render
		<self> ...
```

### Methods that return tags (helper renderers)

**Не используй `return` перед тегами в методах — только implicit return (последнее выражение).**
`return` с переносом строки перед составным тегом не компилируется.

```imba
# ✅ CORRECT — implicit return
def my-card item
	<div.card>
		<p> item.title

# ✅ CORRECT — if/elif/else без return
def badge-for type
	if type == 'win'
		<div [bgc:green]> "WIN"
	elif type == 'loss'
		<div [bgc:red]> "LOSS"
	else
		<div [bgc:gray]> "DRAW"

# ❌ WRONG — return + newline + tag = compile error
def my-card item
	return
		<div.card>
			<p> item.title
```

### Auto-render (periodic updates)

**НЕ используй `schedule(interval: N)` в `def mount`** — не работает в Imba 2.
Правильный паттерн: атрибут `autorender` на `<self>`:

```imba
tag live-counter
	get value
		return Date.now!  # зависит от времени

	<self autorender=250ms>   # перерисовывается каждые 250ms
		<span> value

# Manual commit after async
def fetchData
	data = await api.get!
	imba.commit!
```

### `imba.commit!` — When to Use

**Only after async operations or external callbacks.** Imba auto-commits after user event handlers.

```imba
# ✅ after await
def load
	data = await api.fetch!
	imba.commit!

# ✅ in setTimeout
setTimeout(&, 1000) do
	status = 'done'
	imba.commit!

# ❌ WRONG — @click already triggers commit
<button @click=(do data = !data; imba.commit!)>
```

### Inline Handler Complexity

Keep inline handlers to a single action. Multiple actions → separate method.

```imba
# ✅ single assignment
<button @click=(record.seed = random.string(10))> "Refresh"

# ❌ multiple actions inline
<button @click=(do doSomething!; doAnotherThing!)>

# ✅ extract to method
def handleSave
	doSomething!
	doAnotherThing!

<button @click=handleSave> "Save"
```

### Slots
```imba
<slot> "Default content"
<slot name="header">
<slot name="actions">

# Usage shorthand
<card-layout>
	<"header"> <h2> "Title"
	<p> "Body"
	<"actions"> <button> "Save"
```

### Global & Teleport
```imba
# Click outside + escape key
tag dropdown
	open = false

	<self>
		<global
			@click.outside=(open = false)
			@keydown=(open = false if e.key == 'Escape')
		>
		<button @click=(open = !open)> "Toggle"
		<div.menu [d:block]=open> ...

# Teleport to body (for modals)
tag modal-dialog
	open = false

	<self>
		<button @click=(open = true)> "Open"
		<global>
			if open
				<div.overlay @click=(open = false)>
					<div.modal @click.stop> ...
```

---

## `imba.mount`

Monts tag into `document.body` and enables auto re-render. Without `imba.mount` reactivity doesn't work.

```imba
imba.mount <app>

# Block form
imba.mount do
	<main @mousemove=(x=e.x,y=e.y)>
		<input bind=title>
```

**Multiple parallel mounts:**
```imba
imba.mount <app>
imba.mount <notifications-layer>
imba.mount <modal-layer>
```

**`<global>` vs `imba.mount`** — Use `<global>` for teleport *inside* a component (content removed with component). `imba.mount` — for independent root layers.

---

## Router

```imba
tag app
	<self>
		<nav>
			<a route-to='/home'> 'Home'
			<a route-to='/about'> 'About'

		<[d:box h:100%]>
			<div route='/home'> 'Home page'
			<div route='/about'> 'About page'
			<div route='/*'> '404'

# Features
<div route='/home/'>         # exact match
<div route='/home/*'>        # wildcard
<div route='/user/:id'>      # dynamic segment

# Route methods
tag Genre
	def routed params, state
		data = state.genre ||= await genres.fetch(params.id)
```

---

## Code Patterns

### Dropdown Component Pattern

```imba
tag my-dropdown
	open = false
	items = []

	def select item
		data = item
		open = false

	def toggle
		open = !open

	def close
		open = false

	css self pos:rel d:inline-block w:100%
		.trigger w:100% p:6px 8px rd:6px bgc:#0B0F19 bd:1px solid #1C2332 c:white fs:12px cursor:pointer tween:all 0.15s
			&:hover bgc:#1C2332
		.list pos:abs zi:10000 t:calc(100% + 4px) l:0 bgc:#0B0F19 rd:8px bd:1px solid #1C2332 p:4px d:block w:120px mah:160px ofy:auto shadow:0 8px 24px black/50
			scrollbar-width:thin
			scrollbar-color:#1C2332 #0B0F19
		.item p:6px 8px rd:4px cursor:pointer fs:11px fw:500 tween:all 0.15s
			&:hover bgc:#1C2332
			&.active bgc:#FF8533 c:black

	<self>
		<.trigger @click.stop=toggle> data
		if open
			<global @click.outside=close>
			<.list>
				for item in items
					<.item .active=(data == item) @click.stop=(select(item))> item
```

**Key points:** `pos:abs zi:10000` for list, `mah:` + `ofy:auto` for scroll, `<global>` for teleport under `overflow:hidden`, `@click.stop` to prevent bubbling.

### Conditional + Loop Pattern

```imba
# ❌ BAD — TypeScript thinks 'item' is a number
if loaded
	for item in items
		<.item-box key=item.id ease>

# ✅ GOOD — wrap loop in container element
if loaded
	<.items-list>
		for item in items
			<.item-box key=item.id ease>
```

---

## `<global>` — global listeners with auto-cleanup

Mount a `<global>` element inside any tag to attach listeners to `window`/`document` with **automatic cleanup on unmount**. No `def mount` / `def unmount` boilerplate. Canonical for modals, popovers, dropdowns:

```imba
tag popup
	<self>
		<global @click.outside=close @keydown.esc=close>
		<.modal> 'content'
```

Use this instead of manual `addEventListener` + `removeEventListener` in lifecycle hooks.

---

## `autorender=` — control rerender frequency

Tags rerender on observable change by default. Override with the `autorender` attribute for animation loops / polling:

```imba
<self autorender=60fps>     # animation loop, 60fps
<self autorender=100ms>     # poll every 100ms
<self autorender=1s>        # once per second
```

Replaces `setInterval` + manual `imba.commit`.

---

## `def setup` — lifecycle between props and first render

Runs **after props are assigned** but **before** the first render. The right place to derive state from props. Field initializers run too early (no props yet); `def mount` runs too late (after first render).

```imba
tag editor
	prop record
	def setup
		draft = {...record}     # has access to record, runs before render
```

Order: field init → `setup` → first render → `mount`.

---

## Tag inheritance with getter overrides — the modal pattern

Define a base tag with `get`-based slots; subclasses override the getters. Cleaner than `<slot>` when you have multiple structured insertion points:

```imba
tag basic-popup
	get header
		<h2> 'Default title'
	get body
		<p> 'Override me'
	def close
		imba.unmount self

	<self>
		<global @click.outside=close @keydown.esc=close>
		<.modal>
			<.header> header
			<.body> body
			<button @click=close> 'Close'

tag popup-room < basic-popup
	prop room
	get header
		<h2> (creating ? 'Create room' : "Edit {room.name}")
	get body
		<form> ...
```

Canonical Imba pattern for reusable shells (modals, panels, cards).

---

## `<%semantic-name>` divs — exists, but prefer CSS classes

Imba supports `<%header>`, `<%body>`, `<%footer>` — these compile to a plain `<div>` and exist as a language feature. **But in this project we default to plain CSS classes** (`<.head>`, `<.body>`, `<.footer>`) even when the class has no styles attached to it yet, because:

- Editor syntax highlighting colors `.class` names distinctly, making component structure scannable at a glance
- Classes are familiar to anyone reading the code; `<%name>` is an Imba-specific construct readers pause on
- Classes can pick up CSS later without renaming the element
- The naming works identically — `<.head>` reads just as structurally as `<%head>`

```imba
# ✅ project default — use CSS classes, even unstyled ones
<self>
	<.head> <h2> "Title"
	<.body> ...
	<.footer> <button> "OK"

# ⚠️ valid Imba, but not the project idiom
<self>
	<%head> <h2> "Title"
	<%body> ...
	<%footer> <button> "OK"
```

Keep `<%name>` in mind only as a "this exists" reference; reach for `<.name>` by default.

---

## `dataset` — declarative `data-*` attrs

```imba
tag widget
	def mount
		dataset.payload = 'test'    # → <widget data-payload="test">
		dataset.user-id = user.id   # → data-user-id
```

Useful for analytics hooks and CSS attribute selectors.

---

## Props — just declare fields

To accept props, declare plain fields on the tag. The parent passes them as attributes. **No special syntax, no `prop` keyword, no `defineProps`.**

```imba
tag user-card
	name = ''
	age = 0
	change = null     # callback prop — call as `change!` from inside

	<self>
		<h2> name
		<p> "Age: {age}"
		<button @click=(change && change!)> "Notify parent"

# parent
<user-card name="Ada" age=37 change=on-change>
```

- Any field is reactive — assigning to it from the parent triggers a re-render.
- Callback props are bare fields, called with `!` (e.g. `change!`, `submit!`). Don't reach for "event emitters" — fields *are* the prop API.
- Fields are the **default and primary** way to pass data. Use them for everything except the one specific case below.

## The built-in `data` field

**Every Imba tag already has a `data` field.** You don't declare it, you don't import it, it's just there on `self` from the start, and it can hold anything — a string, an object, an array, `null`. Think of it as a default "payload slot" that comes with every custom element.

```imba
tag user-card
	<self>
		<h2> data.name      # `data` is already there, no declaration
		<p> "Age: {data.age}"

<user-card data={name: 'Ada', age: 37}>
```

This is also what makes `bind=` work on custom elements: writing `<my-field bind=user.name>` is sugar for `<my-field data=user.name>` plus a two-way write-back. Inside the tag you wire it to a real input with `<input bind=data>`, and the chain `inner-input ↔ data ↔ user.name` is automatic.

```imba
tag my-field
	<self>
		<input bind=data>

<my-field bind=user.name>     # bind= writes through `data`
```

### When to use `data` vs named fields

| Situation | Use |
|---|---|
| You want two-way binding from the parent | `data` (via `bind=`) |
| The parent should pass a single value/object that *is* the tag | `data` |
| The parent passes several distinct values, callbacks, or flags | **Named fields** — declare them explicitly |

For the React-style "callback prop" case (`onLogin`, `onSubmit`, `onChange`), declare a plain single-word field on the tag and call it as a function. Don't reach for `data` — that's a different concept.

```imba
tag login-form
	login = null              # callback prop, plain field, single word
	email = ''
	password = ''

	def submit
		await login!({email, password})

	<self>
		<form @submit.prevent=submit> ...

<login-form login=handle-login>
```

**Rule of thumb:** if you're not writing `bind=` on the tag, you're not using `data`. Reach for named fields instead.

---

## `Class.inherited` — static auto-registration hook

A static `inherited` method on a class fires when something extends it. Use for plugin/auto-registration patterns:

```imba
class Page
	static def inherited subclass
		router.register subclass

class HomePage < Page    # → router.register fires automatically
```

---

## `extend class HTMLElement` — fields on every DOM node

```imba
extend class HTMLElement
	get $rect
		getBoundingClientRect!
```

Now any element has `el.$rect`. Use sparingly.
