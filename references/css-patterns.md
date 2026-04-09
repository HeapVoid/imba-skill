---
name: imba-css-patterns
description: "Advanced Imba CSS patterns beyond shortcuts: element-relative units, global style flags, caret parent selectors, color precision and HSL adjustment, semantic divs, value aliases, and responsive tricks."
---

# Imba CSS Patterns

Advanced patterns beyond the shortcut table. For the basic shortcut reference and palette, see `references/css-shortcuts.md`.

## Element-relative units via `@resize.css`

The `@resize.css` modifier exposes the element's own width/height as CSS units (`elw` / `elh`) on itself. This is **container queries without container queries** — children can size themselves relative to *this* element.

```imba
tag panel
	<self @resize.css>
		css s:50px bg:rose4
		# inside any descendant:
		.child fs: calc(1elw / 10)
		# 1elw = 1% of self's width, 1elh = 1% of self's height
```

Custom unit names:

```imba
<self @resize.css(1panel-w, 1panel-h)>
	# now descendants can use 1panel-w / 1panel-h
```

Use this whenever you'd reach for ResizeObserver, container queries, or `clamp(...)` based on viewport.

## Global style flags via `document.flags`

Toggle a flag globally on the document, and any CSS block in any file can react to it via `@`-modifier syntax:

```imba
# anywhere in your code
if state.user
	document.flags.add('mod-logged-in')

# anywhere in any css block
css .nav-item
	d: block @logged-in: none      # hidden when logged in
```

Convention: prefix flag with `mod-`. The `@`-modifier matches without the prefix. This is **the Imba way** to do "is dark mode / is logged in / is mobile / is in PWA standalone" styling — no React Context, no theme provider, no prop drilling.

Combine with media queries to set flags from JS:

```imba
document.flags.toggle('mod-standalone',
	window.matchMedia('(display-mode: standalone)').matches)

global css body
	bg: white @standalone: gray2
```

**Per-element style flags** work the same way — `self.flags.add('dark')` + `global css .dark bg:black`.

## `<%semantic-name>` divs

When you want a meaningful element name without polluting the class namespace, use `<%name>`. Compiles to a plain div but reads like a custom element:

```imba
tag chat
	<self>
		<%header>
			css bg: $base9 p: 16px
			<h2> "Title"
		<%body>
			css fl: 1 ofy: auto p: 16px
			for msg in messages
				<message data=msg>
		<%footer>
			css p: 16px
			<input bind=draft>
```

`<%header>` can't collide with another `.header` class because it's namespaced to the file's compiled output. Use this freely instead of `<.section-foo>` when the name is purely structural, not stylistic. See also `references/tags.md`.

## Color precision and HSL adjustment

The palette has more granularity than the visible `0..9` steps:

```imba
c: blue4
c: blue440        # interpolated between blue4 and blue5
c: blue465
c: blue5
```

And you can shift HSL lightness inline:

```imba
bg: blue3-3       # blue3 minus 3 lightness steps
bg: blue5+2       # blue5 plus 2 lightness steps
```

Useful for hover/active states without defining new tokens: `bg: blue6  bg@hover: blue6+1`.

## Color CSS variables — `#name` vs `$name`

Imba has TWO kinds of CSS variable prefixes:

| Prefix | Purpose |
|---|---|
| `$name` | generic CSS variable (compiles to `var(--name)`) |
| `#name` | **color** CSS variable (gets full color tooling) |

```imba
css #site-bg: gray85
css bg: #site-bg
```

The `#`-prefixed form is preferred for color tokens because Imba's color tooling (HSL adjustment, contrast checking) understands them.

## Caret parent selectors — `^`, `^^`, `^^^`, `..`

A child can react to **the parent's state** without nesting selectors. Caret count = how many levels up:

| Syntax | Means |
|---|---|
| `^@hover` / `^.active` / `^@focus` | Parent (1 level up) is hovered / has `.active` / focused |
| `^^@hover` / `^^.open` | Grandparent (2 levels up) |
| `^^^@hover` | Great-grandparent (3 levels up) |
| `..@hover` / `..@focus` / `...active` | **Any** ancestor up to the root has the state |

Combine with property-modifier syntax for one-line responses:

```imba
tag icon-box
	css self
		.letter
			c: $base7
			c^@hover: $base6      # parent hovered → letter brightens
			c^.active: $blue5     # parent has .active → letter is brand color
			tween: ease-in-out .2s
```

The `..` form (any ancestor) is useful for global theming hooks: `c..@dark: warm1` means "if any ancestor matches dark mode, this color is warm1".

**Specific parent up the chain:**

```imba
^.collapsed       # direct parent has .collapsed
^^^.collapsed     # parent's parent's parent has .collapsed
c^@hover: blue    # text turns blue when DIRECT parent is hovered
```

**Any ancestor** (`..` walks all the way to root):

```imba
o..collapsed: 0   # opacity 0 if .collapsed is anywhere in the ancestor chain
c..@dark: warm1   # text color responds to ANY ancestor in dark mode
```

## `@`-modifier collapse

Multiple `@`-modifiers on one selector collapse into a single combined state — no nesting needed:

```imba
css @hover @focus @checked
	# matches when ALL three states are active simultaneously
```

If you want them as separate alternatives, nest:

```imba
css @hover
	*@focus
```

## Structural pseudos inside `css self`

`@nth-of-type(N)`, `@nth-child(N)`, `@first-child`, `@last-child` work directly inside `css self` blocks:

```imba
css self
	bg: teal3
	@nth-of-type(1) bg: blue3
	@nth-of-type(2) bg: teal3
	@nth-of-type(3) bg: purple3
```

Plus `&.covered bg: red4` for self-class state — `&` is the current element. Use these instead of generating per-index classes from JS.

## `light-dark()` — native CSS theming

```imba
css c: light-dark(gray9, gray0) bg: light-dark(white, gray9)
```

Native CSS function — works without `@dark` modifier or flags. Browser handles it via `color-scheme`.

## `>val` / `<val` min/max shorthand

In a CSS prop, prefix the value with `>` for **min** and `<` for **max**:

```imba
w: >24       # min-width: 24
w: 100px <80vw    # width:100px max-width:80vw
h: 80% >100px     # height:80% min-height:100px
```

⚠️ Marked experimental in `undocumented.md` — may be removed. Prefer `mih`/`mah`/`miw`/`maw` for stability.

## Value aliases — `pos:abs`, `rd:full`, `fs:sm`

`pos:abs` → `position:absolute`, `pos:rel` → `position:relative`, `pos:fix` → fixed, `pos:stk` → sticky. Caveat: these are compile-time substitutions, so you can't interpolate them dynamically.

`rd:full` → `border-radius: 9999px` (perfect circles).

Font size also has aliases: `fs:sm`, `fs:md`, `fs:md-`, `fs:lg`, `fs:xl`, `fs:xs` etc.
