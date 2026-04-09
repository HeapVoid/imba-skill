---
name: imba-syntax
description: "Core Imba syntax: functions, conditionals, loops, ranges, operators, naming conventions, imports/exports. Read this when writing or translating any non-trivial Imba code."
---

# Imba Syntax Reference

## Functions & callbacks

```imba
# Standard definition
def greet name
	"Hello, {name}!"

# Default params
def add a, b = 0
	a + b

# Arrow-style for callbacks (NEVER use `=>`)
let double = do(x) x * 2

# Calling — parens optional
greet "Alice"

# No-args call — `!` is idiomatic for side-effecting calls
Math.random!
update!

# Callbacks
items.filter do(item) item.active
items.reduce(&, 0) do(sum, item) sum + item.weight

# Shorthand positional args ($1, $2, ...)
items.filter do $1 > 3

# Browser globals — always prefix with `window.`
cloned = window.structuredClone(original)
res = await window.fetch('/api')
```

The `&` placeholder in `items.reduce(&, 0)` is Imba's way of saying "the callback goes after the initial value" — useful when the callback would otherwise look ambiguous.

## Conditionals

```imba
if score > 90
	"Excellent"
elif score > 70
	"Good"
else
	"Keep going"

unless ready
	console.log "not yet"

# Postfix
console.log "too much!" if amount > max
return null unless user?

# Ternary — both forms work
result = if condition then "yes" else "no"
result = condition ? "yes" : "no"
```

## Loops & iteration

```imba
for item in items
	console.log item

# With index
for item, i in items
	console.log "{i}: {item}"

# Filter inline
for item in items when item.active
	console.log item

# Map (the for-loop returns an array)
let names = for user in users
	user.name

# Continue with a value, break with a value
for num in [1,2,3,4,5]
	continue -1 if num == 3
	num * 2
# → [2, 4, -1, 8, 10]

# while / until
while condition
	doSomething!

# Object iteration — `own` skips inherited keys
for own key, value of obj
	console.log "{key}: {value}"
```

### Ranges and slices

```imba
[0 .. 5]    # 0,1,2,3,4,5  inclusive
[0 ... 5]   # 0,1,2,3,4    exclusive

arr[1 .. 3]    # inclusive slice
arr[2 ..]      # from index 2 to end
arr[.. -1]     # everything except the last
```

**Spaces around `..` are mandatory** — without them the parser reads `1.` as a float literal and breaks.

## Operators

| Operator | Meaning |
|---|---|
| `&&` / `and` | logical AND |
| `\|\|` / `or` | logical OR |
| `!` / `not` | logical NOT |
| `??` | nullish coalescing |
| `is` / `===` | strict equality |
| `isnt` / `!==` | strict inequality |
| `isa` | type check (`x isa 'string'`, `x isa Array`) |
| `?` | existence check (`if user?`) |
| `..` | safe navigation (`obj..property`) — Imba's `?.` |
| `...` / `*x` | spread |
| `?=` | default assignment (assign if currently nullish) |
| `\|\|=` | assign if currently falsy |
| `=?` | change-aware assignment — returns `true` only if the value actually changed (useful inside reactive code to avoid redundant work) |

## Naming conventions

| Thing | Convention | Example |
|---|---|---|
| Local variables | single word preferred | `state`, `update`, `user` |
| Components (custom tags, exported) | kebab-case **with a dash** | `language-selector`, `todo-app` — must contain `-` (browser custom-element rule) |
| Local tags (file-scoped) | PascalCase | `Box`, `UserCard`, `Counter` |
| ⚠️ Avoid one-word lowercase tag names | — | `tag counter` is ambiguous: not kebab (no dash), not PascalCase. Use `Counter` or `counter-x` |
| Side-effecting calls | `!` suffix | `update!`, `commit!`, `close!` |
| Private fields | `#` prefix | `#cache`, `#connected` |
| Boolean getters / methods | `?` suffix | `get empty?`, `get ready?`, `def valid?` |

**One concept = one word — this is a HARD project rule.** All fields, methods, getters, and locals must be a single word. Multi-word names belong **only to tags** (which are required to contain a dash). The visual payoff: when you read code, anything composite is a tag, anything single-word is data or behavior. The boundary is unmistakable.

```imba
# ❌ wrong — multi-word names on fields/getters/locals
tag todo-app
	doneCount = 0
	done_count = 0
	done-count = 0     # all three are wrong
	get isValid do ...
	def addItem item
		...

# ✅ right — single-word names everywhere except the tag itself
tag todo-app
	todos = []
	draft = ''
	get done                # the count of done todos
		todos.filter(do(t) t.done).length
	get valid?
		draft.trim!.length > 0
	def add
		todos.push {text: draft, done: no}
```

When you find yourself reaching for a two-word name, take it as a signal to:
1. **Rename** to a single more accurate word (`doneCount` → `done`, `userName` → `name` on a `user` object, `addItem` → `add`).
2. **Group** related state into one object: instead of `userName` + `userEmail`, write `user = {name: '', email: ''}`.
3. **Group related getters** into one getter returning an object literal — see "Grouped getters" below.
4. **Split the entity** — if neither rename nor group works, the tag is doing too much.

Deviating from this rule marks code as foreign immediately.

**Dashed identifiers — careful with subtraction.** `let my-var = 1` is legal, but `number-items` is parsed as identifier `number-items`, not `number - items`. Put spaces around `-` whenever subtraction is intended.

## Imports & exports

```imba
# Exports — `export` keyword is REQUIRED for cross-file access
export const utils = { ... }
export class MyClass
export def login email, password
export def register email, password, data = {}
export default tag app

# Imports — local files MUST include `.imba` extension
import { db, lex } from './index.imba'
import { login, register } from './api.imba'
import './components/button.imba'   # side-effect import
import type { User } from './models.imba'

# npm packages — no extension needed
import { notify } from 'imba-notifications'
```

Without `export`, a `def`/`class`/`tag`/`const` is **file-private** and any `import` referring to it will fail at build time. Without the `.imba` extension, the bundler can't find local files.

## Code patterns

### Cloning state safely

```imba
data = window.structuredClone(app.active)
```

Use this when you need a defensive copy before mutating — Imba reactivity tracks references, so mutating the original would trigger reactions you may not want.

### `isa` — instanceof AND typeof in one operator

```imba
[] isa Array              # instanceof
'hello' isa 'string'      # typeof (string literal RHS = typeof check)
7 isa ('string' or 'number')   # composable
```

Always prefer `isa` over `instanceof` / `typeof` in Imba code.

### `try` / `if` as expressions

```imba
let obj = try JSON.parse(input) catch e {}
let label = if user then user.name else 'anon'
```

One-line safe parse, no temporary variable, no mutation.

### Spread shorthand `*x`

```imba
let xs = [1, 2, 3]
console.log(*xs)          # equivalent to ...xs
fn(*args)
```

Same semantics as `...`, two characters shorter.

### `$0` — the arguments object

`$1`, `$2` etc. are positional parameter shorthands. `$0` is the whole `arguments` object:

```imba
def variadic
	console.log $0        # the arguments array-like
```

### `def` inside object literals

Don't write `add: do(a,b) ...`. Write:

```imba
let obj =
	def add a, b
		a + b
	def sub a, b
		a - b
```

Same syntax as inside a class. Reads cleaner.

### `||=` lazy init

```imba
get cache
	#cache ||= new Map
```

Standard JS, but worth remembering — replaces `if !#cache then #cache = new Map`.

### `super` and `super.method!`

In an inheriting tag/class, `super!` calls the parent's same-named method. `super` (no parens) forwards `...args` automatically. `super.other-method!` calls a different parent method.

```imba
tag fancy-button < base-button
	def click
		super!                  # parent's click first
		analytics.track 'click'
```

### Class constructors accept a property bag

```imba
class Note
	content = 'hello'

let n = new Note({content: 'world'})    # sets instance fields from the object
```

Useful for hydrating from `JSON.parse` results.

### Group related getters into one getter returning an object literal

When you find yourself writing several two-word getters about the same subject (`get loaded?`, `get rendered?`, `get visible?`), that's a signal to **group them under one getter that returns an object literal** with scalar AND method keys:

```imba
# ❌ flat — two-word names, no visual grouping
get image-loaded? do img.complete and img.naturalWidth > 0
get image-rendered? do image-loaded? and self.offsetParent
get image-visible? do image-rendered? and not hidden

if image-loaded? and image-visible?
	...

# ✅ grouped — one getter returning an object literal
get image
	loaded?: img.complete and img.naturalWidth > 0
	rendered?: img.complete and self.offsetParent
	visible?: not hidden
	move: do(dx,dy) self.x += dx; self.y += dy

if image.loaded? and image.visible?
	image.move(10,0)
```

Scalar/computed keys (`loaded?: expr`) are evaluated each access. `key: do(...) ...` keys are callable methods. Read at the call site as `image.loaded?` (no parens — it's a property) and `image.move(10,0)`. Two-word names splitting into two scope levels makes the entity boundary explicit.

### Grouped namespace (object as module)

```imba
export const random =
	range: do(min, max, base = undefined)
		Math.round(random.from(base) * (max - min) + min)
	from: do(base)
		return crypto.hash(base) / 4294967296 if base isa 'string'
		Math.random!
```

A common Imba pattern: instead of exporting many small top-level functions, group related helpers under one `const` and export the namespace. Lets you call `random.range(0, 10)` from anywhere.
