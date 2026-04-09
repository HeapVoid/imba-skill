# Imba CSS Shortcuts & Basics

Scopes, inline styles, the full shortcut table, the built-in palette, basic selectors, media queries, variables, transitions/ease, grid.

---

## ⚠️ Three rules you will break on a large component

These are the mistakes that reappear on any component bigger than a todo list. Read them before writing CSS.

1. **NEVER use hex colors.** Not `#2c3e50`, not `#e74c3c`, not `#fff`. Use palette tokens: `gray8`, `red5`, `white`, `blue6`, `$spot4`. Hex literals mark code as alien — the project ships a palette and you must use it. Full palette below. The ONLY exception is `#` inside `url(...)` fragments.

2. **Interpolation `{...}` works ONLY in inline styles `[...]`, NEVER in `css` blocks.** For anything dynamic (grid-template, computed width, color from a lookup), put it in an inline style bracket on the element:
   ```imba
   # ❌ broken — {COLS} is a literal string inside a css block
   css .board
       gtc: repeat({COLS}, 32px)

   # ✅ inline style on the element
   <div.board [gtc:"repeat({COLS}, 32px)"]>

   # ❌ broken — `css` is not a tag attribute
   <div.cell css c:{colors[n]}>

   # ✅ inline style bracket
   <div.cell [c:{colors[n] or 'inherit'}]>
   ```
   The `css` keyword introduces a *block* at tag-body or file level. On an element, dynamic styles go in `[...]`.

3. **Use shortcuts, not longform, for the common table.** `cur:pointer` not `cursor:pointer`. `bgc:` not `background-color:`. `ff:` not `font-family:`. The shortcut table is the idiom; longform is a fallback for properties the table doesn't cover.

---

> **Advanced patterns** (element-relative units via `@resize.css`, global style flags via `document.flags`, caret parent selectors `^`/`^^`/`..`, color precision `blue440` / HSL `blue3-3`, `<%semantic>` divs, value aliases `pos:abs`/`rd:full`/`fs:sm`, `>val`/`<val` min-max shorthand, structural pseudos inside `css self`): see `references/css-patterns.md`.

> **Official docs:** https://github.com/imba/imba.io/tree/master/content/docs
> - `style-syntax.md` — полный синтаксис стилей
> - `style-modifiers.md` — модификаторы
> - `style-units.md` — кастомные единицы измерения
> - `style-properties.md` — список всех свойств

---

## Scopes

```imba
# Global — applies everywhere in the app
global css
	body
		margin: 0

# File-scoped — only applies to elements in THIS file
css .btn
	display: block
css button
	bg: #b2f5ea  # only affects <button> in same file

# Component-scoped — only affects literal elements inside this tag
tag my-button
	css self
		bgc: $spot4
		bc: $spot4
		c: $font9
	# css self styles are scoped to <my-button> and its literal children

# Inherited scoped styles — styles propagate through inheritance
tag base-item
	css d:block m:2 p:3 bg:gray2
	css h1 fs:lg fw:600 c:purple7
tag pink-item < base-item
	css bg:pink2  # inherits css from base-item, overrides bg
```

**Key rules:**
- Styles declared inside a `tag` only affect **literal** elements in that tag
- Nested components (`<app-item>`) are NOT affected by parent's scoped styles
- Global styles (`global css`) apply everywhere
- File-scoped styles (`css selector`) only apply to elements in the same file

---

## Inline Styles

```imba
<self [d:flex ja:center p:16px bgc:#151B2B]>
<.icon [c:{colors[name]}]>

# Conditional
<div[p:2 c:green9] [td:s c:gray4]=item.done>

# Pseudo-states
<div[o:0.5 o@hover:1]>
<div[p:3rem @lg:5rem p@print:0]>
```

---

## Conditional CSS Classes

```imba
# Single conditional class — use .className=(expression)
<button .active=(tab == 'chat')> "Chat"
<div .done=(task.status == 'done')> "Task"

# Multiple conditional classes on same element
<div .self=(is-self) .other=(!is-self)>

# In loops
for color in colors
	<span.dot .selected=(active-color == color) [bgc:{color}]>

# Combined with static classes
<span.tab .active=(tab == 'tasks') @click=select('tasks')> "Tasks"
```

**Key rules:**
- `.className=(expr)` — adds class when expression is truthy, removes when falsy
- **Not** `:class={name: expr}` — that syntax does not exist in Imba 2
- Multiple `.class1=(a) .class2=(b)` on same element — all independent
- If value is a simple boolean getter/field, parens are optional: `.done=is-done` or `.done=!is-done`
- Parens required for comparisons and expressions: `.active=(tab == 'chat')`, `.done=(task.status == 'done')`

---

## Scoped CSS Block

```imba
tag my-component
	css self
		d:flex fld:column
		g:16px
		bgc:#0F1621

		.button
			bgc:#FF8533
			@hover bgc:#FF9955
```

---

## Nesting & Selectors

```imba
css .panel
	bgc:#151B2B
	.title          # → .panel .title (descendant)
		fs:18px
	&.active        # → .panel.active (same element)
		shadow:0 4px 20px black

# Nested selectors — everything before first property is a selector
css .card
	display: block
	background: gray1
	.title color:blue6     # matches .card .title
	h2.desc color:gray6    # matches .card h2.title
	&.large padding:16px   # matches .card.large
```

**Deep selectors (escape scoped confines):**
```imba
# >>> — penetrates shadow DOM / nested components
css div >>> p c:blue6    # matches ALL <p> inside div, even in nested components

# >> — direct children only (like > but also non-literal children)
css div >> p c:blue6     # matches only immediate <p> children
```

**Parent/ancestor selectors** (`^`, `^^`, `..`) — see `references/css-patterns.md`.

**Class modifiers:**
```imba
css button c:white bg.primary:blue
# is the same as:
css button c:white
    .primary bg:blue
```

**`:not()` — используй `@not` вместо `:not`:**
```imba
.card.accordion@not(.active) .strip
	flex-basis:34px   # → .card.accordion:not(.active) .strip
# ❌ :not(.active) — не работает, ошибка компиляции
# ✅ @not(.active) — правильный синтаксис Imba
```

---

## Shorthands — полный список

Источник истины: `node_modules/imba/src/compiler/styler.imba`, `export const aliases`. Ниже — полная таблица из компилятора (на момент Imba v2). Если шорткат для свойства существует — **используй его всегда**. Длинная форма в продакшен-коде Imba встречается только когда шортката нет.

### ⚠️ Don't invent shortcuts — check the table

LLMs (and humans coming from Tailwind) tend to **guess** shortcuts and produce things that don't exist in Imba. Common invented shortcuts that **DO NOT WORK**:

| ❌ Invented | ✅ Correct | Property |
|---|---|---|
| `max-w`, `min-w`, `max-h`, `min-h` | `maw` `miw` `mah` `mih` | min/max width/height |
| `rdd` | `rd` | border-radius |
| `flg`, `flex-g` | `fl` | flex (`fl:1` = `flex:1`) |
| `ls` | `lst` | list-style |
| `cursor` | `cur` | cursor (longform also works) |
| `font` | `ff` (family), `fs` (size), `fw` (weight) | no single `font` shortcut |
| `input@`, `button@` | `input`, `button` (just the tag name) | nothing — `@` is event/state, not a CSS suffix |
| `bgcolor` | `bgc` or `bg` | background-color |
| `txt-align` | `ta` | text-align |
| `fdir` | `fld` | flex-direction |
| `jc-content` | `jc` | justify-content |

**Rule:** if you're not sure a shortcut exists, **use the long-form CSS property** (it always parses) or check the table below. Never invent.

### ⚠️ THE most common shortcut mistake: min/max width/height

> **`miw` / `mih` / `maw` / `mah`** — *that's* the shape. `m` + (`i`nner|`a`) + (`w`|`h`).
>
> Do NOT write `min-w` / `max-w` / `min-h` / `max-h`. Those look right because Tailwind uses them, but Imba doesn't recognize them as shortcuts — they get emitted as unknown CSS properties and silently do nothing. If your `max-h:300px` has no effect, this is why.
>
> Long-form (`max-height: 300px`) parses but is non-idiomatic. Use the shortcut.

### Layout & box

| Short | Full | Short | Full |
|---|---|---|---|
| `d` | display | `pos` | position |
| `w` | width | `h` | height |
| `s` | **size** (= w + h) | `o` | opacity |
| `miw` | min-width ⚠️ NOT `min-w` | `mih` | min-height ⚠️ NOT `min-h` |
| `maw` | max-width ⚠️ NOT `max-w` | `mah` | max-height ⚠️ NOT `max-h` |
| `t` | top | `b` | bottom |
| `l` | left | `r` | right |
| `zi` | z-index | `of` | overflow |
| `ofx` | overflow-x | `ofy` | overflow-y |
| `ofa` | overflow-anchor | `tof` | text-overflow |
| `pe` | pointer-events | `us` | user-select |
| `ws` | white-space | `cur` | cursor (also accepts full word) |

### Padding / margin

| Short | Full | Short | Full |
|---|---|---|---|
| `p` | padding | `m` | margin |
| `pt`/`pr`/`pb`/`pl` | padding-top/right/bottom/left | `mt`/`mr`/`mb`/`ml` | margin-top/right/bottom/left |
| `px` | padding-left + padding-right | `py` | padding-top + padding-bottom |
| `mx` | margin-left + margin-right | `my` | margin-top + margin-bottom |

### Flex

| Short | Full |
|---|---|
| `fl` | flex |
| `flf` | flex-flow |
| `fld` | flex-direction |
| `flb` | flex-basis |
| `flg` | flex-grow |
| `fls` | flex-shrink |
| `flw` | flex-wrap |

### Justify / Align / Place

| Short | Full | Notes |
|---|---|---|
| `ji` | justify-items | |
| `jc` | justify-content | |
| `js` | justify-self | |
| `ai` | align-items | |
| `ac` | align-content | |
| `as` | align-self | |
| `ja` | **justify-align** | combined — `ja:center` = ai+jc center |
| `jai` | place-items | |
| `jac` | place-content | |
| `jas` | place-self | |

### Grid

| Short | Full | Short | Full |
|---|---|---|---|
| `g` | gap | `cg` | column-gap |
| `rg` | row-gap | `gtr` | grid-template-rows |
| `gtc` | grid-template-columns | `gta` | grid-template-areas |
| `gar` | grid-auto-rows | `gac` | grid-auto-columns |
| `gaf` | grid-auto-flow | `ga` | grid-area |
| `gr` | grid-row | `gc` | grid-column |
| `gt` | grid-template | `grs`/`gre` | grid-row-start/end |
| `gcs`/`gce` | grid-column-start/end | `gcg`/`grg` | grid-column-gap/row-gap |

### Background

| Short | Full | Short | Full |
|---|---|---|---|
| `bg` | background | `bgc` | background-color |
| `bgp` | background-position | `bgi` | background-image |
| `bgr` | background-repeat | `bgs` | background-size |
| `bga` | background-attachment | `bgo` | background-origin |
| `bgclip` | background-clip | | |
| `c` | color | | |

### Border

| Short | Full | Short | Full |
|---|---|---|---|
| `bd` | border | `rd` | border-radius |
| `bdt`/`bdr`/`bdb`/`bdl` | border-top/right/bottom/left | `bdx`/`bdy` | border-x / border-y |
| `bs`/`bst`/`bsr`/`bsb`/`bsl`/`bsx`/`bsy` | border-style (sides) | | |
| `bw`/`bwt`/`bwr`/`bwb`/`bwl`/`bwx`/`bwy` | border-width (sides) | | |
| `bc`/`bct`/`bcr`/`bcb`/`bcl`/`bcx`/`bcy` | border-color (sides) | | |
| `rdtl`/`rdtr`/`rdbl`/`rdbr` | individual corners | `rdt`/`rdb`/`rdl`/`rdr` | side-pair corners |
| `bxs` | box-shadow | | |

### Outline

| Short | Full |
|---|---|
| `ol` | outline |
| `olo` | outline-offset |
| `olc` | outline-color |
| `ols` | outline-style |
| `olw` | outline-width |

### Typography

| Short | Full | Short | Full |
|---|---|---|---|
| `ff` | font-family | `fs` | font-size |
| `fw` | font-weight | `lh` | line-height |
| `ls` | letter-spacing | `ta` | text-align |
| `va` | vertical-align | `tt` | text-transform |
| `td` | text-decoration | `tdl`/`tdc`/`tds`/`tdt` | decoration line/color/style/thickness |
| `tdsi` | text-decoration-skip-ink | `tuo` | text-underline-offset |
| `te`/`tec`/`tes`/`tep`/`tet` | text-emphasis (parts) | `txs` | text-shadow |

### Transform

| Short | Full |
|---|---|
| `x`, `y`, `z` | translate components |
| `rotate` | rotate |
| `scale`, `scale-x`, `scale-y` | scale |
| `skew-x`, `skew-y` | skew |
| `origin` | transform-origin |

### Easing / Transitions

| Short | Full |
|---|---|
| `tween` | transition (the one to use) |
| `ea`/`ead`/`eaf`/`eaw` | ease-all (and -duration/-function/-delay) |
| `es`/`esd`/`esf`/`esw` | ease-styles (+ d/f/w) |
| `eo`/`eod`/`eof`/`eow` | ease-opacity (+ d/f/w) |
| `ec`/`ecd`/`ecf`/`ecw` | ease-colors (+ d/f/w) |
| `eb`/`ebd`/`ebf`/`ebw` | ease-box (+ d/f/w) |
| `et`/`etd`/`etf`/`etw` | ease-transform (+ d/f/w) |

### Pseudo-content

| Short | Full |
|---|---|
| `prefix` | content (rendered as `::before`) |
| `suffix` | content (rendered as `::after`) |

**`d:` flex shortcuts** — `d:` accepts compound values that expand to full flex configs:
- `d:vcc` = `display:flex; flex-direction:column; align-items:center; justify-content:center`
- `d:hcc` = same but horizontal
- `d:vflex` / `d:hflex` — flex column / row, defaults
- `d:vtl`, `d:hbr`, etc. — direction + corner alignment

**Flex shortcuts:**
- `d:vcc` — vertical center
- `d:hcc` — horizontal center
- `d:hflex` — horizontal flex
- `d:vflex` — vertical flex
- `d:vtl`, `d:hbr` — top-left / bottom-right

### ⚠️ Частые ошибки в shorthands

| ❌ Неправильно | ✅ Правильно | Почему |
|---|---|---|
| `min-w:` | `miw:` | Shorthand: **m** + **i** + **w** (min-width) |
| `max-w:` | `maw:` | Shorthand: **m** + **a** + **w** (max-width) |
| `min-h:` | `mih:` | Аналогично: **m** + **i** + **h** |
| `max-h:` | `mah:` | Аналогично: **m** + **a** + **h** |
| `box-shadow:` | `bxs:` | **bx** + **s**hadow |
| `cursor:` | `cur:` | **cur**sor |
| `border:` | `bd:` | **b**or**d**er |
| `opacity:` | `o:` | **o**pacity |
| `margin-bottom:` | `mb:` | Аналогично: `mt:`, `ml:`, `mr:` |
| `padding-right:` | `pr:` | Аналогично: `pt:`, `pl:`, `pb:` |
| `margin-top:` | `mt:` | Аналогично: `mb:`, `ml:`, `mr:` |
| `text-align:` | `ta:` | **t**ext **a**lign |

**Запомни паттерн min/max:**
- `miw` = min-width, `maw` = max-width
- `mih` = min-height, `mah` = max-height
- НЕ `min-w`, `max-w`, `min-h`, `max-h` — это НЕ shorthands, а полные CSS свойства

**`px:`, `py:`, `mx:`, `my:` — это ВАЛИДНЫЕ shorthands:**
- `px: 14px` = padding-left: 14px + padding-right: 14px
- `py: 8px` = padding-top: 8px + padding-bottom: 8px
- `mx: auto` = margin-left: auto + margin-right: auto
- `my: 16px` = margin-top: 16px + margin-bottom: 16px

---

## Built-in color palette

Imba ships a full color system inlined into the compiler — see `imba.io/docs/css/colors`. The palette names are **literal hsla values inlined at compile time**, NOT CSS variables, which has important consequences for opacity (see below).

**Tone scales** (each has steps `0..9`, light → dark):

```
warm     warmer     gray     cool     cooler
red      orange     amber    yellow   lime
green    emerald    teal     cyan     sky
blue     indigo     violet   purple   fuchsia
pink     rose       slate    zinc     stone     neutral
```

Plus `black`, `white`, `transparent`, `clear`. Use them everywhere instead of hex literals:

```imba
css .card
	bg: warm9
	c: warm1
	bd: solid 1px warm8
	bxs: 0 4px 12px black/30
```

### Slash opacity — only on built-in literals, NOT on CSS variables

Imba lets you append `/N` to a built-in color name to set the alpha channel:

```imba
bg: blue6/40        # ✅ blue6 at 40% opacity
c: black/10         # ✅
bd: solid 1px white/20   # ✅
```

This works **because the compiler inlines `blue6` as a literal `hsla(...)` and substitutes the alpha at parse time**. The slash form does NOT work on CSS-variable tokens you define yourself:

```imba
global css html
	$base1: warm1
	$base8: warm8
	$brand: blue6

css .x
	bg: $base8/50    # ❌ doesn't work — $base8 is var(--base8) at runtime
	c: $brand/40     # ❌ same problem
```

**The correct pattern** (used in real Imba projects) — define a *new* token at the desired opacity using the literal, then use the new token plain:

```imba
global css html
	$base8: warm8
	$base85: warm8/50    # ✅ alpha applied to the literal at compile time
	$brand: blue6
	$brand-dim: blue6/40 # ✅

css .x
	bg: $base85          # ✅ no slash, just the token
	c: $brand-dim
```

If you need a one-off translucent color in a single rule, use the literal directly: `bg: blue6/20`. Don't try to slash a `$token`.

---

## Selectors — pseudo-states, parents, and modifiers

### `@`-modifier syntax (the primary form)

Imba uses `@hover`/`@active`/`@focus` etc. as **single-line modifiers on a property**, in addition to the nested `&:hover` form:

```imba
css .btn
	bg: blue6
	bg@hover: blue7        # one-property hover, inline
	c: white c@hover: warm0
	tween: ease-in-out .2s
```

This is denser than the nested form when you only have one property responding to the state. All `@`-modifiers can stack on the same line.

**Common mistake — modifier in the wrong position:**

```imba
# ❌ wrong — `@hover:bg:gray1` is not valid Imba CSS
css li
	@hover:bg:gray1

# ✅ right — modifier goes AFTER the property name
css li
	bg@hover: gray1
```

The pattern is `<prop>@<state>: <value>`, never `@<state>:<prop>:<value>`. The nested-block form (`@hover` as a selector) also works but the inline form is more concise for single-property reactions.

Common modifiers: `@hover`, `@active`, `@focus`, `@focus-visible`, `@focus-within`, `@disabled`, `@checked`, `@first-child`, `@last-child`, `@odd`, `@even`, `@placeholder`, `@before`, `@after`, `@empty`, `@target`, `@root`, plus media: `@700` (min-width), `@!650` (max-width), `@dark`, `@light`, `@portrait`, `@landscape`, `@hover-none`, `@motion`, `@!motion`.

### Parent-state selectors

`^`, `^^`, `^^^`, `..` — see `references/css-patterns.md` for the full treatment.

### `@important`

`@important` is a *prefix* on a property, not a suffix:

```imba
.type @important pl: 8px    # ✅
.type pl: 8px @important    # ❌
```

---

## Pseudo-selectors & @important (legacy form, still valid)

```imba
css .btn
	bgc:blue
	@hover bgc:darkblue
	@active bgc:navy
	@first-child fw:bold

# @important — must come BEFORE properties, not after!
# ✅ Correct:
.type @important pl:8px
# ❌ Wrong:
.type pl:8px @important

# Combined in inline
<button[bg:gray2 @hover:gray3 @focus:blue]>
```

---

## Media Queries

```imba
css div@700 display:block      # min-width 700px
css div@!650 display:none      # max-width 650px
css div width:100% @700:80% @1300:1200px
```

---

## CSS Variables & Units

```imba
# CSS variables — use $ prefix (NOT --)
css div $varname:100px width:$varname

# Inline style with CSS variable
<div [$arrow-left:23px]>

# Custom units
global css @root
	1space: 14px
	1col: calc(100vw / 12)

css .field
	p: 1space
	w: 12col
```

---

## Transitions & Animations (`ease`)

The `ease` attribute on a tag enables managed DOM enter/exit. Imba keeps the element in DOM until all transitions complete.

```imba
# Simple fade
<div[o@off:0] ease> "fades in/out"

# Directional enter/exit
<div[y@in:100px y@out:-100px] ease> "slides in from below, out above"

# Custom duration
<div[o@off:0 y@off:50px ease:400ms] ease> "slow fade+slide"

# Nested elements also animate
<div[o@off:0 ease:600ms] ease>
	<div[x@in:-200px x@out:200px]> "slides horizontally"
	<span[scale@off:0]> "scales in/out"
```

**Animation state modifiers:**

| Modifier | Description |
|----------|-------------|
| `@off` | Styles when element is outside DOM (both enter and exit) |
| `@in` | Initial styles on **enter** |
| `@out` | Final styles on **exit** |
| `@enter` | CSS class while entering |
| `@leave` | CSS class while leaving |

```imba
# Comes from below, leaves above
<div.alert[y@in:100px y@out:-100px] ease> "message"

# ease as CSS property (duration)
<div[o@off:0 ease:1s] ease> "1 second fade"
```

**With `key=` on lists — required for per-element enter/exit animation:**
```imba
for item in items
	<.card key=item.id [o@off:0 y@off:20px] ease>
		<span> item.name
```

---

## Keyframes

```imba
# Global animation — available everywhere
global css @keyframes blink
	0% c:white
	100% c:blue

# Non-global — only available in current file
css @keyframes fade-in
	from opacity:0
	to opacity:1

# Override animations in selector (powerful for customization)
global css @keyframes blink
	0% c:white
	100% c:blue

# Override just inside #header .item
css #header a
	@keyframes blink
		from opacity:0
		to opacity:1
```

**Using animations:**
```imba
css a
	@hover animation: blink 2s
```

---

## CSS Grid Syntax

```imba
# grid-template-columns/rows — values separated by SPACES
# ✅ CORRECT
& d:grid gtc:46px 1fr cg:16px
& d:grid gtc:1fr 2fr 1fr rg:8px
& d:grid gtc:repeat(3, 1fr)

# ❌ WRONG — underscore instead of space
& d:grid gtc:46px_1fr        # compiles to "46px_1fr" (invalid CSS)
```

```imba
# Grid column/row span — MUST have spaces around /
# ✅ CORRECT
.item gc: 1 / -1   # spans all columns
.item gr: 1 / 3    # rows 1 to 3

# ❌ WRONG — no spaces around /
.item gc:1/-1
```

---

## One-Line vs Multi-Line

```imba
# Short — one line
css .tab d:flex fld:column ai:center gap:6px p:10px rd:8px cursor:pointer

# Longer — multi-line
css .panel
	bgc:#151B2B
	rd:12px
	of:hidden
	bd:1px solid #1C2332
```
