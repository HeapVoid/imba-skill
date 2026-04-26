# Imba Data Binding Reference

Form inputs, `bind=`, checkbox arrays, multiselect, range, custom elements with bind.

> Source: https://github.com/imba/imba.io/blob/master/content/docs/data-binding.md — this file mirrors the official examples plus a few extras.

---

## Cheat sheet — read this first

| Input type | What `bind=x` does |
|---|---|
| `text` / `textarea` / `password` / `email` | Two-way string binding to `x` |
| `number` / `range` | Two-way **numeric** binding (auto-coerced from string) |
| `checkbox` (bool) | `bind=flag` toggles `flag` between true/false |
| `checkbox` (array) | `bind=arr value=item` adds/removes `item` from `arr` on toggle |
| `radio` | `bind=choice value=item` sets `choice = item` when checked |
| `select` (single) | `bind=choice` — `<option value=x>` sets `choice = x` |
| `select multiple` | `bind=arr` — selected `<option>` values become `arr` |
| `<button bind=x value=y>` | Acts like a checkbox: clicking sets `x = y`, gets `.checked` class when `x == y`, clicking again toggles |
| Custom element `<my-tag bind=foo>` | Writes to a magic field named **`data`** on the tag (see below) |

**The hard rules:**

1. **`bind=` is two-way.** Don't pair it with `@input=` to write back manually — `bind=` does that already. `@input` is only for *side effects* (e.g. validation pings).
2. **Bound `value=` literals must be primitives or stable references.** For arrays/objects, use `value=item` (the same object reference each render), not `value={...item}`.
3. **`bind=path.to.field` works.** `bind=user.name`, `bind=options.width`, `bind=person.interests` — Imba reads/writes through the dotted path. No need to wrap in a getter/setter.
4. **`bind=arr[i]`** also works for indexed access.
5. **`<button bind=...>` is the segmented-control pattern.** Don't write your own toggle group with `@click` + class logic — use `bind` and style `.checked`.
6. **For custom elements, the magic prop name is `data`.** `<my-field bind=user.name>` sets `self.data = user.name` on `<my-field>`. Inside `my-field` you write `<input bind=data>` and the chain `inner-input → data → user.name` is automatic. Don't try to use any other field name — it must be exactly `data`.
7. **Checkbox-array binding requires `value=`.** Without `value=`, the checkbox can't know what to add/remove from the array.
8. **No `bind=` on `<form>` itself.** Bind individual inputs.
9. **`bind=` works inside loops** without keys — the path is per-iteration.

---

## Text Inputs

```imba
let message = "Hello"

<self>
	<input type='text' bind=message>
	<label> "Message is {message}"
```

## Textarea

```imba
let tweet = ''

<self>
	<textarea bind=tweet placeholder="What's on your mind...">
	<label> "Written {tweet.length} characters"
```

## Number

```imba
let a=1
let b=2

<self.group>
	<input type='number' min=0 max=100 bind=a/> " + "
	<input type='number' min=0 max=100 bind=b/> " = "
	<samp> "{a + b}"
```

## Range Slider

```imba
let point = {r:0, x:0}

<self>
	<input type='range' bind=point.x>
	<input type='range' bind=point.r>
	<label> "{point.x},{point.r}"
```

---

## Checkbox — Single Boolean

```imba
let bool=no

<self>
	<label>
		<input[mr:1] type='checkbox' bind=bool />
		<span> "Enabled: {bool}"
```

### ✅ Todo-item pattern: `bind=` does the toggle, `<label>` makes the row clickable

A common mistake is to put `@click=(todo.done = !todo.done)` on the parent `<li>` and then `@click.stop` on the checkbox to prevent double-toggle. **Don't.** The checkbox already has two-way binding via `bind=todo.done`. To make the whole row clickable, wrap it in a `<label>` — clicking anywhere on a `<label>` activates its child input for free, no JS needed:

```imba
# ✅ idiomatic — label handles row click, bind handles toggle
for todo in todos
	<label.row .done=todo.done>
		<input type='checkbox' bind=todo.done>
		<span> todo.text

# ❌ wrong — manual @click + @click.stop fight each other
for todo in todos
	<li .done=todo.done @click=(todo.done = !todo.done)>
		<input type='checkbox' bind=todo.done @click.stop>
		<span> todo.text
```

## Checkbox — Array (Multi-Select)

Checkbox bound to an array adds/removes its `value` from the array:

```imba
let options = ['React','Vue','Imba','Svelte','Ember']
let interests = []

<self>
	<div>
		for option in options
			<label[mr:2]>
				<input type='checkbox' bind=interests value=option/>
				<span[pl:1]> option
	<label> "Interested in {interests.join(', ')}"
```

**How it works:**
- When checkbox is checked → `value` is pushed to `interests` array
- When unchecked → `value` is removed from `interests` array

---

## Radio Buttons

```imba
let options = ['React','Vue','Imba','Svelte','Ember']
let interest = 'Imba'

<self>
	<div>
		for option in options
			<label[mr:2]>
				<input type='radio' bind=interest value=option/>
				<span[pl:1]> option
	<label> "Interested in {interest}"
```

---

## Select — Single

```imba
let options = ['React','Vue','Imba','Svelte','Ember']
let focus = 'Imba'

<self>
	<select bind=focus>
		for item in options
			<option value=item> item
	<label> "Focused on {focus}"
```

## Select — Multiple

```imba
let projects = [{name: 'Project A', color: 'blue'}, ...]
let choices = []

<self>
	<select multiple bind=choices>
		for item in projects
			<option value=item> item.name
	<label> "Selected:"
	<div>
		for item in choices
			<div[color:{item.color}]> item.name
```

---

## Button Bind

Buttons bound to data behave like **checkboxes**. Clicking toggles the checked state.

```imba
let state = 'one'
css button.checked bxs:inset bg:gray2 o:0.6

<self>
	<div.group>
		<button bind=state value='one'> "one"
		<button bind=state value='two'> "two"
	<label> "State is {state}"
```

**Key points:**
- A `checked` class is automatically added when value matches
- Multiple clicks toggle (like checkbox)
- Useful for segmented controls / toggle groups

---

## Custom Elements with Bind

Custom elements can participate in `bind=` by implementing `get data` / `set data`:

```imba
let options = {
	width: 12
	height: 12
	title: 'Something'
}

tag Field
	get type
		typeof data == 'number' ? 'range' : 'text'

	<self[d:flex js:stretch]>
		<label[w:80px]> <slot> 'Field'
		<input[flex:1] type=type bind=data>

imba.mount do <section>
	<Field bind=options.title> 'Title'
	<Field bind=options.width> 'Width'
	<Field bind=options.height> 'Height'
	<label> "{options.title} is {options.width}x{options.height}"
```

**How it works:**
- `bind=` sets the `data` property on the element
- Custom element uses `bind=data` on its internal input
- Changes flow through the chain: `input → data (on Field) → options.title`

---

## Complex Form Pattern

```imba
let people = [{name: 'Jane Doe', interests: ['Imba']}]
let person = people[0]

def addPerson
	person = {name: 'John', interests:[]}
	people.push(person)

def addInterest e
	person.interests.push e.target.value
	e.target.value = ''

	<self>
		<header>
			<select bind=person>
				for item in people
					<option value=item> item.name
			<button[ml:2] @click=addPerson> 'Add'
		<article>
			<label[ta:left]> "Editing {person.name}:"
			<input bind=person.name placeholder="Name...">
			<input placeholder="Add interest..." @keyup.enter.prevent=addInterest>
			<div.tags>
				for item in person.interests
					<button[mr:1].chip bind=person.interests value=item> item
```

---

## Code Patterns

### Form with validation state

```imba
tag LoginForm
	email = ''
	password = ''
	errors = {}

	def validate
		errors = {}
		errors.email = 'Required' unless email.includes('@')
		errors.password = 'Min 6 chars' if password.length < 6
		errors

	def submit
		return if validate
		# proceed...

	<self>
		<input type='email' bind=email placeholder="Email">
		if errors.email
			<p [c:red]> errors.email

		<input type='password' bind=password placeholder="Password">
		if errors.password
			<p [c:red]> errors.password

		<button @click=submit> "Login"
```

### Dynamic form fields

```imba
tag DynamicForm
	fields = [
		{key: 'name', type: 'text', label: 'Name'}
		{key: 'age', type: 'number', label: 'Age'}
		{key: 'active', type: 'checkbox', label: 'Active'}
	]
	data = {}

	<self>
		for field in fields
			if field.type == 'checkbox'
				<label>
					<input type='checkbox' bind=data[field.key]/>
					<span[pl:1]> field.label
			else
				<label>
					<span> field.label
					<input type=field.type bind=data[field.key]>
```

### Checkbox group with "Select All"

```imba
tag TodoList
	let allTodos = ['Buy milk', 'Walk dog', 'Code']
	let selected = []

	get allSelected
		selected.length == allTodos.length

	def toggleAll
		selected = (if allSelected then [] else [...allTodos])

	<self>
		<button @click=toggleAll> (if allSelected then "Deselect All" else "Select All")
		for todo in allTodos
			<label>
				<input type='checkbox' bind=selected value=todo/>
				<span[pl:1]> todo
```
