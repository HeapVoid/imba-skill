---
name: imba
description: "Use this skill whenever the user is working with Imba — the language with .imba files, tag-based components, indentation-significant syntax, and built-in reactivity. Trigger on ANY mention of: writing/editing/debugging .imba files, Imba tags (`<div>`, `<self>`), `bind=` form binding, `@click` or `@touch` event handlers, `def`/`tag`/`css` keywords, `imba.mount`, `@observable`, the `bimba` CLI, or projects that import from `imba`. Also trigger when the user pastes Imba code or asks to translate JS/React/Vue to Imba. Imba's syntax differs sharply from JS (tabs only, no arrow functions, no explicit tag closing, ranges need spaces) so consulting this skill prevents almost-certain compile errors. Don't wait for the user to say the word 'Imba' — if a file ends in .imba or the code uses `<self>` or `def`+indentation, that's enough."
---

# Imba Language Skill

Imba looks like Python-flavored JS but it is **its own language with its own compiler**, and small syntactic mistakes (a stray `=>`, `[1..5]` without spaces, an explicit `</div>`) become hard parse errors. Read the Critical Rules below before writing or editing any Imba code, and pull in the relevant reference file for the area you're working in.

## ⚠️ Imba is NOT React, and NOT Tailwind

The most common failure mode when an LLM writes Imba is **importing concepts and naming from other ecosystems** that look similar but are wrong here.

**Imba is not React.** There are no `props`, no `useState`, no `useEffect`, no hooks, no `onClick`, no JSX expressions in `{}`, no fragments, no `children` array, no `forwardRef`, no context. **Tag fields ARE the prop API** — every field on a tag is reactive by default, and the parent passes them as ordinary attributes (`<my-tag count=5 on-change=fn>`). Callback "props" are plain single-word fields you call as functions (`change!`, `submit!`). React-style names like `onLogin`, `onClick`, `onChange` should become idiomatic Imba names (`login`, `click`, `change`) — and even the camelCase capitalization is wrong in Imba (single-word lowercase only).

**Imba's `data` field is NOT a React props object.** `data` is a built-in field that exists on every tag, used together with `bind=` for two-way binding. If you find yourself writing `data.onLogin` or `data.something` to access a "prop", **stop** — that's the React pattern. In Imba you declare a named field (`login = null`) and the parent passes it directly (`<login-form login=handler>`).

**Imba CSS is not Tailwind.** Tailwind class names (`max-w-md`, `min-h-screen`, `text-sm`, `bg-blue-500`, `px-4`, `flex-1`) **do not exist in Imba** and shortcuts that look similar (`max-w`, `min-h`, `flex-g`) are not real Imba shortcuts either. Imba's shortcuts come from a fixed table in the compiler — `maw`, `mih`, `fs`, `bg`, `px`, `fl`. **If you're guessing a shortcut, you're wrong.** Either look it up in `references/css-shortcuts.md` or write the longform CSS property name (`max-width: 32em` parses fine and is better than an invented `max-w:32em` which silently does nothing).

**Quick translation table for the most-stolen-from ecosystems:**

| What you'd write in… | …becomes in Imba |
|---|---|
| React: `function Comp({onLogin})` | `tag comp` with `login = null` field |
| React: `useState(0)` | `count = 0` (any field is reactive) |
| React: `useEffect(() => {}, [])` | `def mount` |
| React: `onClick={handler}` | `@click=handler` |
| React: `<div className="x">` | `<div.x>` |
| React: `{cond && <X/>}` | `if cond` then `<x>` (block, not expression) |
| Tailwind: `max-w-md` / `max-w:32em` | `maw:32em` (or longform `max-width:32em`) |
| Tailwind: `min-h-screen` | `mih:100vh` |
| Tailwind: `text-sm font-bold` | `fs:sm fw:bold` |
| Tailwind: `flex-1` | `fl:1` |
| Tailwind: `rounded-md` | `rd:md` |
| Tailwind: `cursor-pointer` | `cur:pointer` (or `cursor:pointer` longform) |
| Tailwind: `hover:bg-gray-100` | `bg@hover: gray1` |
| Tailwind: `px-4 py-2` | `px:4 py:2` |
| Tailwind: `gap-2` | `g:2` |

If you catch yourself reaching for a name that "feels right" because of React or Tailwind muscle memory, **assume it's wrong** and check the relevant reference file.

## ⚠️ The `?` suffix rule — getters and fields ONLY

**Never write `def something?` on a method.** The `?` suffix is reserved for things you read without calling — **getters** and **fields** (local `let` variables or tag fields). The moment you put `?` on a `def`, you create a name that cannot be called:

```imba
# ❌ WRONG — def with ? and an argument
def member? user
	members.some do(m) m.id == user.id

# The call site is unparseable:
if member?(user)     # ← `?(` collides — compile error
```

```imba
# ✅ CORRECT — either a zero-arg getter …
get empty?
	members.length == 0

if list.empty?      # no parens, reads naturally

# … or a method without the ? suffix
def has user
	members.some do(m) m.id == user.id

if has(user)        # normal call
```

**The rule in one line:** if it takes arguments, it's a `def` and it does NOT end in `?`. If it ends in `?`, it's a `get` or a field and it takes no arguments.

**Good names for boolean-returning methods with arguments:** `has`, `includes`, `contains`, `owns`, `can`, `allows`, `matches`. Never `member?`, `valid?`, `ready?` on a `def`.

If you catch yourself typing `def ` followed by a word ending in `?`, **stop and rename** before continuing. This is one of the most common failure modes and it breaks the build every single time.

---

## ⚠️ Imba is compiled — the compiler will catch you

Everything in a `.imba` file — the tag templates, the `css` blocks, the shortcuts, the indentation, the operators — is **compiled by the Imba compiler before it ever runs**. This is not a permissive scripting environment where "close enough" works. Every invented shortcut (`mnw:0`, `o:none`, `shadow:lg`, `bd-color:...`), every stray `=>`, every `def member?`, every `{}` in a `css` block, every `</div>`, every space-instead-of-tab is a **hard compile error or a silent misparse** that will show up the moment the user runs `bimba`. There is no "the runtime will figure it out".

This means: **when you're unsure, do not guess.** Guessing produces code that looks right and explodes on build. Three hard rules that follow from this:

1. **If you don't know a shortcut for certain, write the longform CSS property.** `max-width:560px` compiles fine; an invented `mxw:560px` or `mnw:0` does not. The style-guide rule "use shortcuts" applies to the ~25 shortcuts you actually know, not to shortcuts you're extrapolating from a pattern. The common ones are listed below — everything outside that list, write longform or check `references/css-shortcuts.md`.
2. **If the task description says "change role" or "load members", that is English prose, not a method name.** Imba method names must still be **one word** (`update`, `load`, `invite`, `remove`). The temptation to mirror the task's phrasing directly into `def change-role` is the #1 source of naming regressions — resist it. The method goes on a tag whose name (`access-popup`) already carries the domain; the method only needs the verb.
3. **If a rule in this file says something is wrong, it's wrong even when the surrounding context makes the wrong form look natural.** The rules exist *because* the wrong form keeps looking natural.

### The ~25 shortcuts you can use without checking

These are the ones that definitely exist. Anything outside this list: **longform or look it up.**

```
d     display            pos   position         zi    z-index
w h s width/height/both  maw   max-width        mah   max-height
miw   min-width          mih   min-height
p m   padding/margin     px py / mx my / pl pr pt pb mx my…
bg bgc background(-color) c     color            ol    outline
bd    border (shorthand) bc    border-color     bw    border-width
rd    border-radius      bxs   box-shadow
fs fw ff font size/weight/family    lh    line-height
ta tt td  text-align/transform/decoration    ls   letter-spacing
d:flex fld flex-direction ai jc ja align/justify/both     fl flex     g gap
d:grid gtc gtr grid-template-cols/rows   g gap
of ofx ofy overflow(-x/-y)    us user-select     pe pointer-events
cur   cursor             o     opacity          tween transition
```

If you find yourself typing `mnw`, `mxw`, `shadow`, `ws`, `te`, `ov`, `bd-color`, `bd:bottom`, `o:none`, **stop** — none of those exist. Write the longform (`min-width:0`, `white-space:nowrap`, `overflow:hidden`, `text-overflow:ellipsis`, `border-bottom:1px solid gray3`, `outline:none`) and move on. Longform always compiles; invented shortcuts always silently fail.

### ⚠️ Directional borders — `bd:bottom` DOES NOT EXIST

This is a persistent regression worth its own callout. There is **no** `bd:top`, `bd:bottom`, `bd:left`, `bd:right` shortcut. `bd:` is the all-sides border shorthand (`bd:1px solid gray3` sets all four sides). For a single side, **always write the longform property**:

```imba
# ❌ WRONG — silently fails, no border rendered
bd:bottom 1px solid gray2
bd:top 1px solid gray2

# ✅ CORRECT — longform
border-bottom:1px solid gray2
border-top:1px solid gray2
```

If you catch yourself typing `bd:` followed by a direction word, **delete and rewrite as longform**. Every time.

### Media queries: nest inside `css`, or use property-level breakpoint modifiers

Two correct patterns. Pick one, never write a top-level `@media` wrapping separate `css` blocks.

**Pattern A — breakpoint modifiers at the property level (idiomatic Imba):**
```imba
css .side
    w:64px                    # base (mobile)
    w@lg:240px                # ≥1024px
    bgc:gray9
```

Imba ships built-in breakpoint state modifiers: `@sm`, `@md`, `@lg`, `@xl` (mobile-first, min-width). Any property can carry one: `fs@md:16px`, `d@lg:grid`, `p@sm:2`. This is the preferred form for 1–2 properties that change at a breakpoint.

**Pattern B — nested `@media` inside a `css` block:**
```imba
css .side
    w:240px
    @media (max-width: 1023px)
        w:64px
        .label d:none
```

Use this when many properties change together at one breakpoint. Note the `@media` is **indented inside** the `css .side` block, not a top-level sibling.

**❌ BROKEN — top-level `@media` with separate `css` blocks under it:**
```imba
# This is NOT a media query grouping — it does not compile as you'd expect.
@media (max-width: 1023px)
    css .side
        w:64px
    css .main
        margin-left:64px
```

## ⚠️ Imba CSS is a compiled DSL — not raw CSS

The `css` blocks in Imba look like CSS but they are **not** CSS — they are compiled by the Imba compiler, which gives the language three superpowers you are expected to use. If your CSS looks like what you'd write in a `.css` file, you're leaving most of the language on the table and your code will read as foreign.

**1. Shortcuts are the idiom, not an optimization.** Imba ships a fixed table of 2–3 letter property names: `bgc`/`bg` (background), `c` (color), `p`/`m` (padding/margin), `w`/`h`/`s` (width/height/both), `mih`/`maw` (min-height/max-width), `fs`/`fw`/`ff` (font size/weight/family), `d` (display), `ai`/`jc`/`ja` (align/justify/both), `fl`/`fld` (flex/flex-direction), `g`/`gtc`/`gtr` (gap/grid-template), `rd` (border-radius), `cur` (cursor), `of`/`ofy` (overflow), `pos`/`t`/`l`/`r`/`b` (position/inset), `us`/`pe` (user-select/pointer-events), `td`/`tt` (text-decoration/transform), `tween` (transition). **Write `cur:pointer`, not `cursor:pointer`. Write `mih:100vh`, not `min-height:100vh`.** Longform is a fallback for properties the table doesn't cover — not a style choice.

**2. The built-in palette replaces hex colors.** Imba ships a full color palette with tokens like `gray1`…`gray9`, `blue1`…`blue9`, `red5`, `green6`, `white`, `black`, plus project tokens like `$spot4`, `$base9`. **Never write hex colors.** Not `#2c3e50`, not `#e74c3c`, not `#fff`. If you need a dark background, it's `gray8` or `gray9`. If you need a warning red, it's `red5`. If you need translucency, it's `black/20` or `blue6/40` — not `rgba(...)`. Hex literals in an Imba file are a giveaway that the author was thinking in raw CSS instead of in Imba, and they will be flagged in review. The ONLY exception is `#` inside `url(...)` fragments.

**2a. Real projects use CSS custom properties — and they just work.** In a real codebase the palette is usually defined as CSS variables (`--color-text`, `--bg-surface`, `--brand-primary`) in a global stylesheet or a `:root` block. **Imba passes `var(--name)` through to the compiled CSS unchanged**, so you can use project variables in both `css` blocks and inline styles without any special syntax:

```imba
# ✅ CSS variables work in css blocks
css .card
	bgc:var(--bg-surface)
	c:var(--text-primary)
	border:1px solid var(--border-default)

# ✅ and in inline styles
<div.card [bgc:var(--bg-surface) c:var(--text-primary)]>

# ✅ you can define them in a global css block in Imba too
global css
	:root
		--text-primary: gray9
		--bg-surface: white
		--brand: blue6
```

The priority order for colors in real code:
1. **Project CSS variables** (`var(--text-primary)`) — if the project has a design system, use it
2. **Imba palette tokens** (`gray9`, `blue6`, `black/20`) — for anything the project variables don't cover, or for prototypes
3. **Hex colors** — never

If you're working in a project with existing CSS variables, **use them** — don't substitute `gray8` for `var(--bg-surface)` because you prefer palette tokens. Match the project.

**2b. One `css self` block per tag, nested — not a dozen top-level `css .foo` siblings.** Put all the component's styles inside a single `css self` block and nest child selectors under it by indentation. Scattering 10+ separate `css .brand`, `css .item`, `css .label` statements across the tag body compiles but reads as noise; the nested form reads as one coherent stylesheet and makes parent/child relationships visible through indentation. Put `&.active`, `&:hover`, `&@hover` modifiers right next to the base selector they modify. Full example in `references/style-guide.md` § "One `css self` block".

**3. Interpolation `{...}` works ONLY in inline style brackets `[...]`, NEVER in `css` blocks.** `css` blocks are compiled once at build time; they can't contain runtime expressions. For anything dynamic, use inline styles on the element:

```imba
# ❌ broken — {COLS} is a literal string inside a css block
css .board
	gtc: repeat({COLS}, 32px)

# ✅ inline style on the element (or hardcode in the css block)
<div.board [gtc:"repeat({COLS}, 32px)"]>

# ❌ broken — `css` is not a tag attribute
<div.cell css c:{colors[n]}>

# ✅ inline style bracket
<div.cell [c:{colors[n] or 'inherit'}]>
```

**The mental model:** think of Imba CSS as **a compiled language with its own vocabulary**, not as a thin wrapper over CSS. Every hex color, every `cursor:pointer`, every `{}` inside a `css` block is a sign you slipped into "raw CSS" mode. Stop, and rewrite it in the Imba vocabulary. Full shortcut table and palette in `references/css-shortcuts.md`.

## ⚠️ Imba is minimalism with deep functionality

**Shorter is better. Always.** A fluent Imba author writes code that is 2–3× shorter than the JS/React equivalent — not because of code golf, but because the language was designed so the idiomatic form *is* the shortest form. Every extra wrapper div, every helper method, every multi-line CSS rule you could collapse, every `@click` you could have been a `bind=`, every explicit `return` before a tag, every field you declared when a getter would do — is a signal that you're not writing in Imba yet, you're writing JS with Imba syntax on top. **If your output is noticeably longer than the reference examples in `references/style-by-example.md`, rewrite it shorter before submitting.**

**Decompose — DRY rules.** "Shorter" applies per-file, not per-project. If a single tag grows past ~15–20 lines of template + CSS for a self-contained unit (a sidebar menu, a modal header, a toolbar, a user row, a card), **extract it into its own tag** — even if it's used only once. The parent becomes a high-level outline, the extracted tag is easy to find and restyle, and its `css self` keeps styles scoped. If you see two sibling blocks with the same shape, extract immediately. Extracting costs ~5 lines of tag boilerplate and pays back every time you read the file. See `references/style-guide.md` § "Decompose repeating blocks" for the full heuristic and a worked sidebar/app-shell example.

Imba's whole philosophy: **the language and the browser already know how to do most things**. Your job is to find the existing mechanism, not to build a new one. Before writing more than 3 lines of imperative interactivity, ask: **is there a built-in HTML element, CSS property, or Imba operator that already does this?** The answer is almost always yes.

The single most-violated example: **don't put `@click` on `<li>`/`<div>` to make a row toggle a checkbox.** Use `<label>` — it automatically routes clicks to the inner input, and `bind=` handles the toggle. Zero event handlers, zero `@click.stop`, accessible by default.

```imba
# ❌ fighting the browser — manual @click + @click.stop to prevent double-toggle
<li @click=(todo.done = !todo.done)>
	<input type='checkbox' bind=todo.done @click.stop>
	<span> todo.text

# ✅ using built-in HTML — <label> routes clicks to its inner input for free
<label.row>
	<input type='checkbox' bind=todo.done>
	<span .done=todo.done> todo.text
```

Same rule for: `<button>` instead of `<div @click>`, `<a href>` instead of `<div @click=navigate>`, `<form @submit.prevent>` instead of `<button @click>` inside a form, `<details><summary>` instead of a custom dropdown with `@click.outside`. **Don't fight `bind=` with manual handlers** — if you have `bind=foo` AND `@click=(foo = !foo)`, delete the `@click`.

Full breakdown in `references/style-guide.md` § "The core principle".

---

## When to load which reference

| You're working on… | Read |
|---|---|
| Basic syntax: functions, loops, conditionals, operators, imports, naming, `isa`, `try`/`if` expressions, `*x` spread, `$0`, object-literal `def`, `\|\|=`, `super`, grouped getters | `references/syntax.md` |
| Tags, components, lifecycle, slots, refs, inheritance, `<global>`, `autorender=`, `def setup`, `dataset`, `Class.inherited`, `extend class HTMLElement` *(note: project defaults to `<.head>`/`<.body>` CSS-class divs over `<%head>`/`<%body>` semantic divs — see tags.md)* | `references/tags.md` |
| CSS shortcuts table, palette, scopes, nesting, inline styles, grid, media, keyframes | `references/css-shortcuts.md` |
| Advanced CSS: `@resize.css`/element units, `document.flags`, caret parent selectors, color precision/HSL, `<%semantic>`, `light-dark()`, value aliases | `references/css-patterns.md` |
| Events: modifiers, `.cooldown`/`.throttle`/`.log`, custom modifiers via `extend class Event`, hotkeys, `@error.trap`, `imba.emit`/`imba.listen`, `def on$name`, per-event scratch state | `references/events.md` |
| Render lifecycle: `mount`, `commit`, `imba.commit`, autorender | `references/rendering.md` |
| Forms: `bind=`, inputs, checkbox arrays, multiselect | `references/data-binding.md` |
| `@observable`, `@autorun`, `@computed`, atomic, spy, `imba.awaits` | `references/reactivity.md` |
| Classes, stores, `extend`, accessors, super | `references/classes.md` |
| `async`/`await`, `Queue` | `references/async.md` |
| `imba.locals` / `imba.session` (local/session storage) | `references/storage.md` |
| `@touch` events, drag, gestures, pointer | `references/touch.md` |
| Bun runtime / `bimba` CLI | `references/bun.md` |
| PocketBase backend hooks | `references/pocketbase.md` |
| Claude Code plugin in VS Code | `references/vscode.md` |
| **Compile/runtime errors — quick fix table** | `references/troubleshooting.md` |
| **Idiomatic Imba writing style — conciseness rules** | `references/style-guide.md` |
| **Worked rewrite examples (navbar, design refactor)** | `references/style-by-example.md` |

When something isn't covered: official docs at imba.io/docs, and the source of truth is `node_modules/imba/src/compiler/` in the user's project — when docs disagree with reality, the compiler wins.

---

## Critical Rules

These are the mistakes that break Imba code most often. Each one has a *reason* — knowing the reason helps you handle edge cases the table doesn't cover.

| Rule | Correct | Wrong | Why |
|---|---|---|---|
| **Indentation = tabs** | `\t` | spaces | Imba is indentation-significant; the lexer rejects spaces and mixed indent silently misparses scopes |
| **Range needs spaces** | `[1 .. 24]` | `[1..24]` | `..` is the safe-navigation operator; without spaces the parser reads `1.` as a float and chokes |
| **Dynamic CSS values in `{}`** | `[bgc: {color}]` | `[bgc: color]` | Inline-style values are parsed as CSS literals unless wrapped — bare identifiers become unknown CSS keywords |
| **Private fields: no space** | `#cache` | `# cache` | `# ` (with space) is a comment. Private-field syntax is the literal token `#name` |
| **Comments: space after `#`** | `# note` | `#note` | Without the space the lexer reads it as a private field reference |
| **No semicolons** | `let x = 1` | `let x = 1;` | Imba uses newline as statement terminator; semicolons cause parse errors |
| **Callbacks use `do`, not `=>`** | `arr.filter do(x) x > 0` | `arr.filter (x) => x > 0` | `=>` is not Imba syntax — `do` is the block-callback form |
| **No `prop` keyword** | `count = 0` | `prop count = 0` | All class fields are reactive by default; `prop` is React, not Imba |
| **Tag attrs: no compound `or`/`and`** | `disabled=!valid?` (define `get valid? do email and password`) | `disabled=!email or !password` | Tag attribute parser doesn't accept compound boolean expressions; extract into a getter |
| **Boolean `?` suffix: ONLY on getters and fields, NEVER on methods** *(see dedicated block below)* | `get valid?`, `let ready? = no` | `def member? user` | See "⚠️ The `?` suffix rule" section below — this is violated constantly, so it has its own block |
| **Event handlers: prefer method reference** | `@click=increment`, `@click=close!` | `@click=do increment!`, `@click=(e) => increment()` (also `=>` is wrong) | The shortest form `@click=method` works for the common case. Use `@click=do(e) ...` only when you genuinely need the event object or multiple statements |
| **Tag names: prefer kebab-case** | `tag todo-app`, `tag user-card`, `tag my-counter` | `tag counter` (one lowercase word), `tag Counter` (PascalCase as a default choice) | Custom HTML tags MUST contain a `-` (browser custom-element rule). **Default to kebab-case for everything** — even single-component files. PascalCase exists only for the rare case where you genuinely need to `import` a tag from another file (`import {Counter} from './counter.imba'`). If the tag is auto-registered (kebab-case), no import is needed and no PascalCase is needed. One-word lowercase names like `tag counter` are always wrong |
| **Browser globals need `window.`** | `window.structuredClone(x)` | `structuredClone(x)` | Imba's TS layer doesn't expose unprefixed browser globals — you'll get "does not exist" errors |
| **Reserved property names** | rename `new`→`creating`, `data`→`payload`, etc. | `new`, `data`, `attr`, `form`, `input`, `class`, `style` as field/method names | These collide with Imba/DOM keywords and either fail to parse or silently misbehave (e.g. `style` overrides the inline-style getter) |
| **Local imports need `.imba`** | `import './a.imba'` | `import './a'` | The bundler requires explicit extension for local files (npm packages are exempt) |
| **`export` is required for `def`/`class`/`const`** | `export def login` | `def login` | Without `export` these are file-private and can't be imported |
| **DON'T `export` kebab-case custom tags** | `tag todo-app` *(no export)* | `export tag todo-app` | Custom-element tags (with a `-` in the name) are auto-registered globally by the Imba runtime — they're available everywhere without import. Exporting them is redundant and marks code as foreign. PascalCase local tags (`tag Counter`) DO need `export` if you want to import them |
| **NEVER write `def render`** | template body directly under `tag` | `def render` wrapping `<self>` | Imba autorenders. Tag template lives at the **top level** of the tag body, not inside any method. `def render` will run as a regular method and the autorender machinery won't see your template. **If you see `def render` in input code, delete the line and dedent its body one level.** |
| **Single-word identifiers for fields/methods/getters** | `count`, `done`, `valid?`, `draft`, `add`, `submit`, `toggle`, `flag` | `doneCount`, `toggle-flag`, `done_count`, `userName`, `isValid` | **The reason this rule exists: in Imba, tag names are the ONLY identifiers allowed to contain a `-`, and they're the ONLY identifiers allowed to be multi-word.** That contrast is load-bearing — when your eye scans code, anything composite (`<todo-app>`, `tag user-card`) is instantly recognizable as a tag/component, and anything single-word (`count`, `done`, `toggle`, `valid?`) is instantly recognizable as data or behavior. Break the rule and you destroy that signal: `toggle-flag` looks like a tag but is a method, and the reader has to pause and reparse every time. Rules with a clear *reason* stick better than rules alone — this is the reason. Two-word methods/fields are always a signal to rename (`toggle`, `flag`), group into one getter returning an object literal (see `references/syntax.md` § grouped getters), or split the entity |

### Never close tags explicitly

```imba
# ✅ correct — indentation closes the tag
<self>
	<div>
		<span> "Hello"
	<p> "Outside div"

# ❌ wrong — </span></div> are parse errors
<self>
	<div>
		<span> "Hello"
		</span>
	</div>
```

Imba tags close based on indentation, the same way Python blocks end. There is no closing-tag syntax in the language at all, so writing `</div>` is not "redundant" — it's a syntax error.

### Static assets: use `const`, never raw strings in `src`

```imba
# ✅
const images = { logo: "./images/logo.png" }
<img src=images.logo>

# ❌ — bundler tries to resolve the literal at parse time
<img src="./images/logo.png">
```

---

## JS → Imba cheat sheet

When the user gives you JS/React code to translate, this is your starting point. For everything beyond simple translation, see `references/syntax.md` and `references/tags.md`.

| JavaScript | Imba |
|---|---|
| `function f(x) {}` | `def f x` |
| `return value` | *(last expression is implicit return)* |
| `this` | `self` |
| `x?.y` | `x..y` |
| `fn()` | `fn!` |
| `typeof x === 'string'` | `x isa 'string'` |
| `x instanceof Array` | `x isa Array` |
| `setTimeout(fn, 500)` | `await wait 500ms` |
| `` `Hello ${x}` `` | `"Hello {x}"` |
| `arr.filter(x => x > 0)` | `arr.filter do(x) x > 0` *(or `for x in arr when x > 0`)* |
| `onClick={() => fn()}` | `@click=fn` *(method ref — shortest form)* |
| `arr.map(x => x.name)` | `for x in arr` → `x.name` |
| `for (let k in obj)` | `for own key, val of obj` |
| `arr.slice(1, 3)` | `arr[1 ... 3]` |
| `[0,1,2,3,4]` | `[0 ... 5]` |
| `class A extends B` | `class A < B` |
| `onClick={fn}` | `@click=fn` |
| `e.preventDefault()` | `@submit.prevent` |
| `useState(0)` | `count = 0` *(plain reactive field)* |
| `useEffect(() => {}, [])` | `def mount` |

---

## Conciseness

For idiomatic code style and conciseness rules, see `references/style-guide.md`.
For worked rewrite examples (153→41-line navbar, 13-file design refactor), see `references/style-by-example.md`.

---

## Workflow rules

- **Always `Read` an Imba file before editing it.** Indentation is load-bearing and you can't reproduce tabs from memory or from your own assumptions about formatting. Reading the file also tells you the surrounding patterns (kebab-case vs PascalCase tags, getter style, etc.) which the project may have established.
- **When you hit a compile error you don't immediately recognize**, check `references/troubleshooting.md` first — most Imba errors have one of a small number of root causes.
- **Imitate the surrounding code's idioms** — naming (single-word locals, `!`-suffix for side effects, `?`-suffix for boolean getters), kebab-case component names, getters for derived values. See `references/syntax.md` § Naming for the full list.

---

## External docs

- [imba.io/docs](https://imba.io/docs) — official
- [github.com/imba/imba](https://github.com/imba/imba) — compiler source
- Context7 MCP: `get-library-docs(libraryID: "/imba/imba", topic: "<topic>")` if available
