# Imba Rendering Reference

Memoized DOM, `imba.mount`, `imba.commit`, `autorender`, manual render, performance.

---

## How Memoized DOM Works

Imba does NOT use a virtual DOM. Tag literals compile to **real DOM elements** with built-in memoization.

```imba
let el = <div.large title="Item">
console.log el # HTMLDivElement (real DOM node!)
console.log el.outerHTML # <div class='large' title='Item'>
```

### Compiled Render Example

```imba
let number = 1
let bool = false

tag App
	<self .ready=bool>
		<h1.header> "Number is {number}"
```

Roughly compiles to:

```js
class App extends HTMLImbaElement {
	render(){
		var $ = (this.cache || this.cache = {});

		// Toggle class only if value changed
		if($.val != bool) this.classList.toggle('ready',$.val = bool)

		if(!$.rendered){
			$.h1 = this.appendChild('h1')      // create & cache
			$.h1.className = "header";          // first render only
			$.h1.insertText("Number is ");      // first render only
			$.t1 = h1.insertText(number);       // create & cache textnode
		} else {
			$.t1.textContent = number;           // only update text!
		}
		$.rendered = true;
	}
};
```

**Key insight:** After first render, re-rendering only updates changed values — not the whole tree.

---

## `imba.mount`

Mounts a tag tree and enables **automatic re-rendering** after every handled DOM event.

```imba
# Block form (most common)
imba.mount do
	<main @mousemove=(x=e.x,y=e.y)>
		<input bind=title>
		<label> "Mouse is at {x} {y}"

# Single element
imba.mount <app>
```

**Without `imba.mount`**, dynamic data does NOT update:

```imba
let array = ["First","Second"]

let view = <main>
	<button @click=array.push('More')> 'Add'
	<ul> for item in array
		<li> item

# ❌ Clicking "Add" pushes to array, but view does NOT update
document.body.appendChild view
```

**With `imba.mount`**, the view updates automatically:

```imba
let array = ["First","Second"]

imba.mount do
	<main>
		<button @click=array.push('More')> 'Add'
		<ul> for item in array
			<li> item
# ✅ View updates on every click
```

### Multiple Mounts

```imba
imba.mount <app>
imba.mount <notifications-layer>
imba.mount <modal-layer>
```

Each mount is an independent reactive tree.

---

## `imba.commit!`

Schedules a re-render for the next animation frame. Only needed for **async operations** or **external callbacks**.

**Imba auto-commits after every handled DOM event.** You rarely need manual commit.

### When to use `imba.commit!`

**✅ After async/await:**
```imba
def load
	data = await api.fetch!
	imba.commit!
```

**✅ In setTimeout:**
```imba
setTimeout(&, 1000) do
	status = 'done'
	imba.commit!
```

**✅ WebSocket message:**
```imba
socket.addEventListener('message', imba.commit)
```

### When NOT to use `imba.commit!`

**❌ Inside event handlers — auto-commits:**
```imba
# ❌ WRONG — @click already triggers commit
<button @click=(do data = !data; imba.commit!)>
```

**❌ After synchronous state changes:**
```imba
def handleClick
	count++
	# ❌ WRONG — no commit needed
	# imba.commit!
```

---

## Manual Render (`el.render!`)

You can manually call `render!` on any element to force re-render.

```imba
tag App
	<self>
		<h1> "Number is {number}"

let el = <App>
document.body.appendChild el

# Manually trigger render
export def update
	number = Math.random!
	el.render!
```

**Rendering cascades to children** — calling `render!` on a parent re-renders all custom children.

```imba
tag Item
	<self> "Random: {Math.random!}"

tag App
	<self>
		<Item>
		<Item>

let el = <App>
el.render!  # Both <Item> components re-render
```

---

## `autorender` — Periodic Updates

For elements that need re-rendering at specific intervals:

```imba
tag Clock
	<self>
		<span> (new Date).toLocaleString!

imba.mount do
	<Clock autorender=1s>
```

**autorender values:**
- `autorender=1s` — every second
- `autorender=500ms` — every 500ms
- `autorender=10fps` — 10 times per second
- `autorender=60fps` — every frame

**Multiple elements with different intervals:**
```imba
imba.mount do
	<app-clock autorender=1s title='New York' utc=-5>
	<app-clock autorender=500ms title='San Fran' utc=-8>
	<app-clock autorender=10fps title='London' utc=0>
	<app-clock autorender=60fps title='Tokyo' utc=9>
```

### Old pattern (DO NOT use)

**❌ `schedule(interval: N)` in `def mount` — doesn't work in Imba 2:**
```imba
tag Clock
	def mount
		schedule(interval: 1000)  # ❌ BROKEN
```

**✅ Use `autorender` instead:**
```imba
tag Clock
	<self autorender=1s>
		<span> (new Date).toLocaleString!
```

---

## Performance Implications

### Why Full Re-render is Fast

In frameworks like React, `render()` creates a **new virtual DOM tree** every time. The reconciler compares old vs new and patches the real DOM.

In Imba, `render()` **modifies the real DOM directly**. After first render, subsequent renders only execute code paths that actually change.

```imba
let number = 1
tag App
	<self>
		<h1.header> "Number is {number}"
		<p> "Static text that never changes"
		<Item>  # Custom component
		<Item>

# First render: creates h1, p, textnodes, Item components
# Second render: only updates number text — everything else is cached
```

### When to worry about performance

- **Expensive computations in render** — use `@computed` getters
- **Filtering large arrays in templates** — cache the result
- **Deep component trees** — each component's render is called

```imba
# ❌ BAD — filters on every render
tag Search
	<self>
		for item in allItems.filter(do $1.includes(query))
			<div> item

# ✅ GOOD — cache with @computed
tag Search
	@observable query = ""

	@computed get filtered
		allItems.filter do $1.includes(query)

	<self>
		for item in filtered
			<div> item
```

---

## Code Patterns

### Async data loading with commit

```imba
tag UserList
	users = []

	def load!
		users = await api.users!
		imba.commit!

	def mount
		load!

	<self>
		for user in users
			<.user> user.name
```

### WebSocket sync

```imba
tag LiveView
	data = {}

	def mount
		imba.ws = new WebSocket('ws://...')
		imba.ws.addEventListener('message', imba.commit)

	def unmount
		imba.ws?.close!

	<self>
		<span> data.value
```

### Custom interval render

```imba
tag Stopwatch
	ms = 0
	#interval = null

	def mount
		#interval = setInterval(&, 10ms) do
			ms++
			imba.commit!

	def unmount
		clearInterval(#interval)

	<self> "{ms / 100}s"
```
