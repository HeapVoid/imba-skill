# Imba Router Reference

Imba has a **built-in client/server router** — no setup needed. It adds `route` and `route-to` properties to elements.

---

## `route-to` — navigation links

Works like `href` but with two extras:
- Enables nested routes
- Automatically adds `.active` class when the linked route is matching

```imba
<nav>
	<a route-to='/home'> 'Home'
	<a route-to='/about'> 'About'
	<a route-to="/user/{user.id}"> user.name
```

---

## `route` — conditional rendering by URL

Elements with `route` **only display** when the URL matches.

```imba
<div route='/home'> 'Home page'
<div route='/about'> 'About page'
```

### Exact routes (trailing `/`)

```imba
<div route='/home/'>   # only /home, NOT /home/anything
```

Without trailing `/`, routes are **wildcard by default** — `/home` matches `/home`, `/home/sub`, etc.

### Explicit wildcard (`*`)

```imba
<div route='/home/*'>  # same as /home (wildcard)
<div route='/*'>       # catch-all fallback
```

### Dynamic segments (`:param`)

```imba
<div route='/user/:id'>        # matches /user/1, /user/abc
<div route='/user/:uid/post/:pid'>  # multiple segments
```

Params available via `route.params`:

```imba
tag user-page
	<self route='/user/:id'>
		<span> "User {route.params.id}"
```

---

## Nested routes

Routes **not starting with `/`** resolve relative to the closest parent route.

```imba
<div route='/home'>
	<a route-to='nested'> 'Go nested'   # → /home/nested
	<div route='nested'>
		'Nested content'
```

---

## Route precedence

**Order matters.** First matching route wins. Put specific routes before wildcards:

```imba
# ✅ correct order
<div route='/items/'>       # exact /items
<div route='/items/new'>    # literal /items/new
<div route='/items/:id'>    # dynamic /items/123
<div route='/*'>            # fallback

# ❌ wrong — /* catches everything, nothing below it renders
<div route='/*'>
<div route='/home'>
```

---

## `def routed` — data loading on route change

Called whenever route params change. Receives `params` and a per-match `state` object for caching.

```imba
tag genre-page
	def routed params, state
		data = state.genre ||= await genres.fetch(params.id)

	<self[o@suspended:0.4]>
		<span> data.title
```

- If `routed` uses `await`, rendering is **suspended** until it completes (element gets `@suspended` state modifier)
- `params` object is **constant per match** — same URL → same object (good for memoization)
- `state` object is **unique per match but persistent** — navigating back reuses the same state

---

## `imba.router`

The application router is always available via `imba.router`.

---

## When to use built-in router vs manual `pushState`

| Pattern | Use |
|---|---|
| **Page switching** — different elements shown for different URLs | Built-in `route` + `route-to` |
| **Data loading** — same layout, different data based on URL | Manual `pushState`/`replaceState` + `popstate` listener, or `def routed` on a container with a dynamic `route` |
| **Mixed** — some page switching + data loading | `route-to` for links (gives `.active` class + proper link behavior), `def routed` or manual `handleUrl` for data |

The built-in router is **element-visibility-driven**: `route` shows/hides elements. For apps where the layout stays the same and only data changes (e.g., selecting a project/chat in a sidebar), manual URL management with `pushState` is often simpler than trying to map variable-length URL segments to `route` patterns.

### Manual URL sync pattern

```imba
class App
	#syncing = false

	def syncUrl replace = false
		return if #syncing
		let path = "/{active.id}"
		if replace
			window.history.replaceState(null, '', path)
		else
			window.history.pushState(null, '', path)

	def handleUrl
		#syncing = true
		const parts = window.location.pathname.split('/').filter(do(p) p.length > 0)
		# ... parse parts, select matching items ...
		#syncing = false
		syncUrl(true)

	def init
		window.addEventListener 'popstate', do handleUrl!
		await handleUrl!
```
