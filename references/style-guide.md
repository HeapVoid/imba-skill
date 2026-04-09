---
name: imba-style-guide
description: "Idiomatic Imba writing rules — how to make code look like Imba instead of transpiled JS. Conciseness principles, component structure, CSS density, naming, and when NOT to build something."
---

# Imba Style Guide

Pure idiomatic-writing rules. For the API reference behind each rule, follow the cross-links. For worked rewrite examples (a 153→41-line navbar, a 13-file design refactor), see `references/style-by-example.md`.

Imba code by experienced authors is **dramatically more compact** than what an LLM tends to produce by default. A real navbar component might be 40 lines; the same logic written verbosely balloons to 150+. This is not a code-golf preference — it's how the language is designed to be used. If your output is 3× the length of the human version, you are wasting tokens and producing code that reads as foreign.

---

## The core principle — minimalism through built-in semantics

Imba is **minimalist on the surface but deep underneath**. Every feature you reach for in JS/React/Tailwind probably already exists either as a one-character CSS shortcut, a built-in HTML behavior, or a single language operator. Your job is to **find the existing thing instead of building a new one**.

This translates into a simple heuristic: **whenever you're about to write `@click` on a non-button, non-link element, stop and ask "is there a built-in HTML/Imba mechanism that already does this?"** Almost always, yes.

### Use HTML semantics, not `@click` on `<div>`

The browser already knows how to do most "interactivity" you'd reach for `@click` to implement. Use the right element and the behavior is free — and accessible, and keyboard-navigable, and screen-reader-friendly, all without writing a line of code.

| You'd write… | Use instead | Why |
|---|---|---|
| `<div @click=submit>` | `<button @click=submit>` | Buttons are focusable, keyboard-activatable, announce as buttons to AT |
| `<div @click=navigate>` | `<a href=url>` | Links work with middle-click, cmd-click, copy-link, right-click menus |
| `<li @click=(item.done = !item.done)>` + `<input type='checkbox' bind=item.done>` + `@click.stop` | `<label> <input type='checkbox' bind=item.done> <span> item.text` | `<label>` makes the whole row clickable for free; `bind=` handles the toggle. No `@click`, no `.stop`, no double-toggle bug. |
| `<div @click=submit>` inside a form | `<form @submit.prevent=submit>` + `<button>` | Hitting Enter in any input submits the form; the submit handler runs once |
| Custom dropdown with `@click.outside` etc | `<details><summary>` + `<menu>` | Native open/close, keyboard, focus trap |
| `<div @click=focus-input>` | `<label for=input-id>` (or wrap input in `<label>`) | Browser routes the click to the input |

**The `<label>` + `bind=` pattern** is the canonical example. You want "click anywhere on a todo row to toggle done." The naive way is `@click=(todo.done = !todo.done)` on the row plus `@click.stop` on the checkbox to prevent double-toggle. The Imba way is **`<label>`**:

```imba
# ❌ — fighting the browser
<li @click=(todo.done = !todo.done)>
	<input type='checkbox' bind=todo.done @click.stop>
	<span> todo.text

# ✅ — using built-in HTML semantics
<label.row>
	<input type='checkbox' bind=todo.done>
	<span .done=todo.done> todo.text
```

`<label>` automatically routes any click inside it to the contained input. `bind=` handles the toggle. There is no event handler in your code at all. **Less code, no bug, accessible by default.**

### Don't fight `bind=`

If you write `bind=foo` on an input AND also `@click=(foo = !foo)` or `@input=(foo = e.target.value)`, **stop**. `bind=` already does both directions. Adding manual handlers either duplicates the work or fights it. The `@input` event modifier is for *side effects* (validation pings, analytics) — not for assignment.

### Don't reach for state when CSS can do it

A `<details>`/`<summary>` doesn't need an `open` boolean. A `<dialog>` has `.showModal()`. `:hover` doesn't need an `isHovered` state. `position:sticky` doesn't need a scroll listener. `aspect-ratio:1` doesn't need `@resize.css`. Reach for the CSS-only or HTML-only solution first; only fall back to Imba state when there's no other way.

### The rule

> **Before writing more than 3 lines of imperative interactivity, ask: is there a built-in HTML element, CSS property, or Imba operator that already does this?** The answer is almost always yes. Imba is "minimalism with deep functionality" — your job is to find the deep thing, not to write a new shallow one.

---

## One `css self` block, not many `css .foo` siblings

**Inside a tag, put all styles in a single `css self` block and nest selectors under it.** Don't scatter a dozen top-level `css .foo` / `css .bar` statements across the tag body. Both compile, but a single nested block reads as one coherent stylesheet for the component — you scan the structure once and see the whole visual identity in one place.

```imba
# ❌ scattered — 10 separate css statements, eye jumps around
tag side-menu
	css self d:flex fld:column w:64px
	css .brand d:flex ai:center px:20px
	css .brand .text ml:10px fs:16px
	css .nav d:flex fld:column g:4px p:12px
	css .item d:flex ai:center px:12px py:10px rd:8px
	css .item.active bgc:blue6 c:white
	css .icon s:24px fs:18px
	css .label ml:12px fs:14px
	css .label@lg d:block

# ✅ one block — nested, reads as one stylesheet
tag side-menu
	css self
		d:flex fld:column w:64px w@lg:240px
		.brand
			d:flex ai:center px:20px
			.text ml:10px fs:16px fw:700
		.nav d:flex fld:column g:4px p:12px fl:1
		.item
			d:flex ai:center px:12px py:10px rd:8px cur:pointer
			&.active bgc:blue6 c:white
			&:hover bgc:gray7
		.icon s:24px fs:18px
		.label ml:12px fs:14px d:none d@lg:block
```

The nested form has three benefits beyond aesthetics:
- **Scope is visually obvious** — everything under `css self` belongs to this component
- **Parent/child relationships are indentation, not long selectors** — `.brand .text` becomes nested `.brand` → `.text`
- **Modifiers sit right next to the base** — `&.active` and `&:hover` under `.item` instead of three separate `css .item`, `css .item.active`, `css .item:hover` lines

Reach for multiple top-level `css` statements **only** when you're styling something outside the tag's own subtree (global styles via `global css`, or file-scoped styles that need to target a different element).

## CSS density

**The biggest single lever: CSS on one line per block.**

```imba
# ❌ verbose — every property on its own line
.tab
	position: relative
	cursor: pointer
	display: flex
	align-items: center
	justify-content: center

# ✅ idiomatic — group with shortcuts
.tab
	pos:rel cursor:pointer d:flex ai:center jc:center
```

Group related properties on one line. The eye reads them as a unit. Long-form is harder to scan, not easier.

## Use theme tokens, never hex

`$base9`, `$blue6`, `$gray3`, `$spot4` — projects ship a design system; consult the project's `css.imba` to see what's available. Hex literals like `#1E1E1E` mark code as alien.

Theme tokens come from Imba's built-in palette — remap to project-semantic names in a `global css html` block. Full palette list in `references/css-shortcuts.md`.

## Use every shortcut

There is almost always a shortcut. Full list in `references/css-shortcuts.md`. Common ones to always reach for: `bgc` `bg` `c` `p` `m` `mxw` `mih` `w` `h` `s` `fs` `fw` `lh` `td` `tt` `bd` `bxs` `rd` `pos` `t` `b` `l` `r` `g` `fl` `fld` `ai` `jc` `cur` `of` `tween` `gtr` `bxs` `us` `pe` `ol` `ja` `as` `tt` `va`.

- **`tween: cubic-in-out .2s`** instead of `transition: all .2s cubic-bezier(...)`.
- **`color/opacity` syntax** for translucency: `black/10`, `$blue6/40` — not `rgba(...)`. Only works on built-in palette literals, not on `$`-tokens.

## Binary class state, not mirrored pairs

Use `.active=(i == current)` and style the *default* (non-active) state. Don't write both `.active=…` and `.inactive=…`. Style for the negative case as well, and you've doubled your CSS.

## Flat structure

Don't decompose into `top-section`/`middle-section`/`bottom-section` if a flat list of children works. Spacing is solved by `g:` and flex, not by wrapper divs.

## No decorative comment blocks

`# ---- Header ----` is noise. Imba code rarely has comments at all; the structure speaks.

## No dead UI

Don't add elements you'll hide with `d:none`. Just leave them out. A `.support-btn { d:none }` rule is a sign you added an element you didn't need.

## Callback props as bare fields

`change` declared as a plain field on a tag is the Imba way to expose a callback prop — call it as `change!` from inside, and the parent passes `<my-tag change=handle-thing>`. No `emit`, no event objects, no subscription.

## CSS Grid for app shells, not flex+overflow hacks

`d: grid gtr: auto 1fr auto g: 16px` gives you header/scrollable-middle/footer in one line — no `pos:absolute bottom:16px`, no `fl:1 mih:0 ofy:auto` chains, no wrapper divs.

## Combined-axis shortcuts

- `s: 48px` = `w:48px h:48px`
- `d: flex ja: center` = `ai:center jc:center`
- `rd: full` = fully rounded
- `aspect-ratio: 1` for parent-sized squares

## Define base typography & buttons globally

Then use semantic HTML in components: `<h2>`, `<p .light>`, `<button .dark>` — not `<span.title>` with custom CSS each time. The browser already has tags for these.

## One-line tags when single child

```imba
<self> <.spinner>
<.icon-box> <user-icon>
<.empty> <simple-loader [s: 48px]>
<.progress> <.bar [w: {(done * 100) / total}%]>
```

Don't break it into two indented lines unless the child has its own children worth indenting.

## Inline computed inline-style for derived geometry

`<.bar [w: {(done * 100) / total}%]>` — no per-width CSS class. `<.active-bg [l:{100/n*i}% w:{100/n}%]>` — no CSS class per position. Better than minting a CSS class for each value.

## Safe navigation in templates

`app.projects.active..id == project.id` — the `..` operator returns `undefined` if the LHS is null/undefined. No `if` guard needed. Use freely in conditional class bindings.

## Decompose repeating blocks into their own tags — DRY matters

Conciseness is not only about shorter lines. **When the same block appears twice, or when a block grows past ~15–20 lines inside a parent, extract it into its own tag.** In Imba this is almost free — a new `tag foo-bar` with a few fields is a dozen lines of boilerplate max — and it pays back immediately:

- The parent becomes readable at a glance (each section is one short tag call)
- The extracted tag is easy to find, easy to restyle, easy to test
- Its CSS is scoped to `css self`, so styles can't leak
- Props are just fields — no ceremony to "pass data down"

**The rule of thumb:** if you're writing a component and you see two sibling blocks that are structurally similar (e.g. a "members list" and a "search results list" that both render user rows with avatars), **extract the shared shape into a tag**. If you see one block that's 20+ lines of template + CSS for a self-contained unit (a sidebar menu, a modal header, a toolbar, a user row, a card), **extract it** even if it's only used once — the parent is easier to read without it, and when the second use-site appears you already have the component.

```imba
# ❌ monolithic — 80 lines of sidebar + main all in one tag
tag app-shell
	items = [...]
	active = 0
	<self.shell>
		<nav.side>
			<.brand> ...
			<.menu>
				for item, i in items
					<.item .active=(active == i) @click=(active = i)>
						<span.icon> item.icon
						<span.label> item.label
			# ...40 more lines of menu CSS...
		<main.main> ...

# ✅ decomposed — each concern in its own tag
tag side-menu
	items = []
	active = 0
	change = null

	css self d:flex fld:column w:64px w@lg:240px ...
	css .item ...

	<self>
		<.brand> ...
		for item, i in items
			<.item .active=(active == i) @click=(active = i; change && change!(i))>
				<span.icon> item.icon
				<span.label> item.label

tag app-shell
	items = [...]
	active = 0
	<self.shell>
		<side-menu items=items active=active change=(do(i) active = i)>
		<main.main> ...
```

The decomposed version is longer in total line count but **each file/tag is shorter and self-contained**, which is the metric that actually matters for readability.

**When NOT to extract:** a one-off 5-line block with no repetition and no internal state is fine inline. Extraction has a small fixed cost (a new tag header, field declarations), so it's only worth it once the block has weight. The heuristic: **≥15 lines OR ≥2 use-sites OR internal state** → extract.

## Don't build features that weren't asked for

A chat-list "create new chat inline form" with 5 helper methods is dead weight if the spec says "list chats". Ship the minimum.

## When two-word names appear, the entity wants to split

Instead of `get image-loaded?` / `get image-rendered?` / `get image-visible?`, group them under one getter that returns an object literal: `get image` returning `{loaded?: ..., rendered?: ..., visible?: ..., move: do(dx,dy) ...}`. Read at call site as `image.loaded?` and `image.move(10,0)`.

Two-word names splitting into two scope levels makes the entity boundary explicit, and **the moment you reach for a third related getter you know it belongs here**. See `references/syntax.md` for the exact object-literal syntax (computed keys + `def` keys).

---

## Name style conventions

These aren't compile errors but they're what makes code look like Imba instead of transpiled JS:

- **One-word names are mandatory** for fields/methods/getters/locals. `state`, `update`, `user`, `valid?`, `done`, `draft`, `add`. **Multi-word names are reserved for tags only** (`<todo-app>`, `tag user-card`). When you read code, anything composite is a tag and anything single-word is data or behavior — that contrast is the whole point. Two words on a field/getter is always wrong; rename, group into an object, or extract a `get name` returning `{loaded?, rendered?, ...}`.
- **Boolean returns end in `?`.** Both for getters (`get ready?`) and method names (`def empty?`). Reads naturally at the call site.
- **Side-effecting methods end in `!`.** `update!`, `close!`, `commit!`. Lets the reader instantly tell "this mutates" from "this returns a value".
- **Prefer explicit `return`** for anything beyond a one-liner. Implicit return is OK for `def double x do x * 2`; for multi-statement methods it's harder to read.
- **Getter names are one word, matching what they return.** `get done`, `get progress`, `get total`. If you reach for `get done-count` or `get doneCount`, drop the suffix — the getter's own name *is* the count. For multiple related stats, extract `get stats` returning `{done, total, progress}`.
- **NEVER write `def render`.** Imba autorenders. The `<self>` template lives at the **top level** of the tag body, not inside any method. If you see `def render` in input code, delete the line and dedent its body one level.
- **Event handlers: shortest form that works.** `@click=method` is the default. `@click=do(e) ...` only when you need the event or multiple statements. Never `=>`.

## Workflow rules

- **Always `Read` an Imba file before editing it.** Indentation is load-bearing.
- **Imitate the surrounding code's idioms** — naming, kebab-case component names, getter style.
- **When you hit a compile error you don't recognize**, check `references/troubleshooting.md` first.

---

## The distilled rule

> Every line of CSS, every wrapper div, every helper method, every comment — make it justify its existence. If a grid row, a global token, a semantic HTML tag, or a one-line shortcut can replace it, it should.
