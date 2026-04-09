---
name: imba-events
description: "Imba event system — modifiers, hotkeys, custom modifiers via `extend class Event`, global event bus (`imba.emit`/`imba.listen`), error boundaries, per-event scratch state, and raw handler control via `def on$name`."
---

# Imba Events

Imba's event system is **modifier-first**: most things that would be imperative state/boilerplate in JS are declarative modifiers chained onto `@click`, `@keydown`, `@submit`, etc.

## Built-in event modifiers

Chain freely — order matters only for `.prevent`/`.stop` which should go first:

| Modifier | Meaning |
|---|---|
| `.prevent` | `e.preventDefault()` |
| `.stop` | `e.stopPropagation()` |
| `.once` | Fire once then auto-remove |
| `.capture` | Register in capture phase |
| `.passive` | Passive listener (no preventDefault) |
| `.self` | Only fire if `e.target === self` |
| `.log('msg')` | Built-in debug modifier — logs when event fires |
| `.debounce(500ms)` | Debounce |
| `.throttle(1s)` | Rate-limit cousin of `.cooldown` |
| `.cooldown(1s)` | Declarative cooldown — see below |
| `.outside` | Only fire for clicks OUTSIDE (with `<global>`) |

```imba
<form @submit.prevent=submit>
<input @keydown.enter=submit>
<button @click.stop=handler>
<button @click.once=handler>
<button @click.prevent.stop.throttle(500ms).log('clicked')=fn>
```

## Declarative cooldown / debounce

```imba
<button @click.cooldown(1s)=submit>
	css @.cooldown bg: gray4
	"Click me"
```

After click, the element gets a `.cooldown` class for 1 second and the handler can't fire again. **Zero state, zero setTimeout, zero refs.** The CSS reacts to the class.

If you omit the duration (`@click.cooldown=async-save`), the cooldown lasts until the handler's promise resolves — perfect for async submits:

```imba
<button @click.cooldown=async-save>
	css @.cooldown o: 0.5
	"Save"
```

Combine: `@click.cooldown.confirm` — both prevent re-fire and require confirmation.

## Custom modifiers via `extend class Event`

Add your own `@modifier-name` to any event by extending the `Event` class. This is the Imba pattern for app-wide concerns like confirm/auth-check/feature-flag-gate:

```imba
extend class Event
	def @confirm msg
		let view = <app-confirm anchor=target template=msg>
		imba.mount view
		view.promise          # the modifier "succeeds" only if this resolves truthy
```

Now anywhere in the app:

```imba
<button @click.confirm('Delete this?')=do-delete> 'Delete'
```

The handler only runs if the confirm modal resolves true. **No event-bus wiring, no callback prop, no hook.**

**Fallback rule:** If a modifier you use isn't defined, Imba falls back to checking for class `mod-name` on the root element. So `@premium=fn` automatically does the right thing if you `document.flags.add('mod-premium')`.

## Raw event registration via `def on$name`

For ultra-rare cases where you need raw control over how a DOM event is registered on a tag:

```imba
tag app-button < button
	def on$click mods, context, handler, options
		# mods = modifier list, options = { capture, once, passive }
		self.addEventListener('click', handler, options)

tag app
	<self>
		<app-button @click.log('clicked')> "log"
```

## Hotkeys

```imba
<self @hotkey('cmd+k')=open-palette>
<self @hotkey('esc').global=close>      # global = catch even when focus is elsewhere
<self @hotkey('*').global>              # swallow ALL hotkeys (modal pattern)
```

## `@error.trap` — error boundaries

Catch errors emitted by descendants:

```imba
<.boundary @error.trap=on-error>
	<risky-component>
```

## Global event bus — `imba.emit` / `imba.listen`

App-wide events, no DOM needed. Use this instead of pulling in EventEmitter / mitt / RxJS:

```imba
imba.emit 'user:login', user
imba.listen 'user:login' do(user) console.log user
```

Any DOM element also has `.emit`:

```imba
some_element.emit('event-name', detail)
```

## Per-event scratch state via `e.$key ||= ...`

Stash arbitrary fields directly on the event object — useful for accumulating state across a touch gesture without a separate `@observable`:

```imba
def draw e
	let path = e.$path ||= new Path2D    # one path per touch, persists across moves
	path.lineTo(e.x * dpr, e.y * dpr)
	$canvas.getContext('2d').stroke(path)
```

The `e.$path ||= new Path2D` idiom: created on first call of the gesture, reused on subsequent moves.

## Touch events (cross-link)

`@touch.fit(self)=draw` — makes `e.x` / `e.y` come back as coordinates **relative to and scaled to** the target element. No `getBoundingClientRect`, no manual `clientX - rect.left`:

```imba
<canvas$canvas[size:{size}px]
	width=size*dpr height=size*dpr
	@touch.fit(self)=draw>
```

`@touch.moved.sync(self)` — one-line drag & drop. Writes back `self.x` / `self.y` as the touch moves:

```imba
<self[x:{x}px y:{y}px] @touch.moved.sync(self)>
```

Constrain axes/distance with `@touch.moved(30px,'x')`. Full details in `references/touch.md`.

## Handler assignment — shortest form that works

```imba
# ✅ CORRECT - function reference (no !)
<input @input=handler>
<button @click=save>
<button @click=save(data)>        # with args
<button @click=(count++)>         # inline expression
<input @keydown=(do(e) e.key == 'Enter' and submit!)>   # e access / multi-statement

# ❌ WRONG - don't call the function
<input @input=handler!>
<button @click=save()>
```

`@click=method` is the default. `@click=do(e) ...` only when you need the event or multiple statements. Never `=>`.
