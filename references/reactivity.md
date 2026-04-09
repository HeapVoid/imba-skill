---
name: imba-reactivity
description: Imba reactive programming — @observable, @autorun, @computed, @lazy, @action decorators, atomic, spy, imba.awaits. MobX-like reactivity built into the language. Includes autorun order, stop-observing, and differences from MobX.
---

# Imba Reactivity

MobX-like reactivity implemented as decorators. Tree-shaken when not used.

## Core Decorators

| Decorator | Purpose |
|-----------|---------|
| `@observable` | Property — marks as reactive source |
| `@autorun` | Method — re-runs when any used observable changes |
| `@computed` | Getter — cached, recomputes only when dependencies change |
| `@lazy` | Getter — evaluated once, then cached forever |
| `@action` | Method — batch multiple observable updates, single autorun/recompute |

## Progression Example: search filter

### Inefficient (rerenders on every render)
```imba
let query = ""
let words = ["apple", "orange", "strawberry"]
tag app
	<self>
		<input bind=query>
		for word in words.filter(do $1.includes(query))
			<div> word
```

### Manual (explicit update call)
```imba
let query = ""
let words = ["apple", "orange", "strawberry"]
let filtered = words
tag app
	def search
		filtered = words.filter do $1.includes(query)
	<self>
		<input bind=query @input=search>
		for word in filtered
			<div> word
```

### @autorun (auto-tracks dependencies)
```imba
let words = ["apple", "orange", "strawberry"]
let filtered = words
tag app
	@observable query = ""
	@autorun def search
		filtered = words.filter do $1.includes(query)
	<self>
		<input bind=query>
		for word in filtered
			<div> word
```

No `@input` needed — `search` re-runs whenever `query` changes, from anywhere.

### @computed (value, not side-effect)
```imba
let words = ["apple", "orange", "strawberry"]
tag app
	@observable query = ""
	@computed get filtered
		words.filter do $1.includes(query)
	<self>
		<input bind=query>
		for word in filtered
			<div> word
```

## Reference

### @autorun
- Runs immediately after instantiation (class) or mount (tag)
- Tags: automatically disposes on unmount
- Does NOT call `imba.commit` automatically
- Supports delay modifier: `@autorun(delay: 100ms)`

**Stopping observation:**
```imba
@observable one = ''
@observable two = ''
@autorun def my-func stop-observing
	one
	stop-observing! if stop-observing
	two  # won't trigger this autorun after stop-observing! called
```

**Order:** autorun functions run in declaration order.

### @computed
- Cached on first use
- Recomputes only when observable dependencies change
- No intermediate renders for unchanged values

### @lazy
- Evaluated once, returns cached value forever

### @action
- Batch multiple observable mutations
- Prevents intermediate `@autorun` calls and `@computed` cache invalidations during the block

## atomic

Batch multiple `@observable` mutations — single re-render, no intermediate `@autorun` calls.

```imba
import { atomic } from 'imba'

# Without atomic: 3 potential re-renders
app.status = 'loading'
app.progress = 0
app.error = null

# With atomic: 1 re-render
atomic do
	app.status = 'loading'
	app.progress = 0
	app.error = null
```

**Example: form reset**

```imba
import { atomic } from 'imba'

tag my-form
	@observable name = ''
	@observable email = ''
	@observable error = null
	@observable submitting = false

	def reset
		atomic do
			name = ''
			email = ''
			error = null
			submitting = false

	def submit
		atomic do
			submitting = true
			error = null
		try
			await fetch('/api/submit', {method: 'POST'})
			reset!
		catch e
			atomic do
				submitting = false
				error = e.message
```

**Rules:** only affects `@observable` properties; `@autorun` and `@computed` cache invalidations fire once after the block.

## spy

Low-level debug tool — run a function and track which observables it reads.

```imba
import { spy } from 'imba'

let state = new class
	@observable count = 0
	@observable name = 'test'

spy(state) do |ctx|
	console.log state.count   # access is tracked

# Full signature: spy(target, fn, context = null)
spy(myObject, do
	myObject.someObservable
	myObject.anotherValue
, optionalContext)
```

Useful for debugging why `@autorun` or `@computed` doesn't react — verify it actually reads the observable.

> Production code: prefer `@autorun` and `@computed` directly.

## `imba.awaits` — wait on observable settling

Pause execution until an observable-dependent callback returns truthy:

```imba
await imba.awaits do user.first_name is 'updated'
```

The callback re-runs every time any observable it touched changes; the await resolves the first time it returns truthy. Use this in tests or imperative flows that need to wait on state.

## Differences from MobX
- No output comparisons on `@computed` (rendering already memoized)
- Language-level integration enables: observable subclasses, autorunning methods that auto-dispose
