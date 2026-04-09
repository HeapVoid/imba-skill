---
name: imba-classes
description: "Imba classes, inheritance, constructors, accessors, super, extend, and state management with @observable/@autorun. Use when working with Imba class-based reactivity, stores, or advanced OOP patterns."
---

# Classes & State Management

## Classes

```imba
class Rect
	width = 10
	height = 10

	constructor w, h
		width = w
		height = h

	get area
		width * height

	set area size
		width = height = Math.sqrt size

	def expand hor, ver = hor
		width += hor
		height += ver

	static def build side
		let item = new self
		item.width = item.height = side
		item

# Inheritance
class Dog < Animal
	def speak
		super.speak!
		console.log "{name} barks"

# Extend native types
extend class Node
	def #append node
		appendChild(node)
```

### Constructor object shorthand

Передать объект в конструктор — свойства установятся автоматически:

```imba
class Note
	content = 'hello'

let note = new Note({content: 'world'})
let copy = new Note(JSON.parse(JSON.stringify(note)))
```

### Accessors

Делегировать get/set поля объекту с методами `$get` / `$set`:

```imba
class Uppercase
	def $set value, target, key
		target[key] = value
	def $get target, key
		target[key].toUpperCase!

let upcase = new Uppercase

class User
	name as upcase

let user = new User
user.name = "sindre"
console.log user.name  # SINDRE
```

### super

```imba
tag modal
	def test
		console.log "hello"

tag my-modal < modal
	def test
		super!          # вызов с нулём аргументов
		super           # вызов с ...args (implicit spread)
		super.setup!    # вызов другого метода родителя
```

---

## State Management

Весь state management в Imba основан на классах.

```imba
# Plain reactive fields (внутри тегов)
tag counter
	count = 0
	<button @click=(count++)> count

# @observable — class-level reactivity
export class Store
	@observable #value = 0
	get value do #value
	set value val
		#value = val

# @autorun — reactive side-effects
@autorun def colorize
	# Re-runs whenever @observable dependency changes
	const hue = (random.from(#seed) * 0.618) % 1
	self.style.setProperty('--chip-base', "hsl({360 * hue}, 70%, 20%)")
```

**Key rules:**
- `@autorun` tracks dependencies at runtime
- Do NOT use `imba.commit!` inside `@autorun`
- Plain fields в тегах — реактивны автоматически
- `@observable` — для классов-хранилищ (Store)
