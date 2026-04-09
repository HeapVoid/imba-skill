---
name: imba-troubleshooting
description: "Quick-fix reference for Imba compile and runtime errors. Read this when an Imba file fails to compile, throws an unexpected error, or behaves in surprising ways. Most Imba errors trace back to a small set of root causes — check here before debugging deeper."
---

# Imba Troubleshooting

When an error doesn't look familiar, scan this table first. The cause is almost always one of these.

## Compile errors

| Error message | Likely cause | Fix |
|---|---|---|
| `Unexpected 'IDENTIFIER'` after a property | Reserved word used as field/method name | Rename. Reserved: `new`, `data`, `attr`, `form`, `input`, `class`, `style`. E.g. `new` → `creating`, `data` → `payload` |
| `Unexpected 'INDENT'` inside a `tag` definition | `css` block missing selector | Use `css self` or `css &` to scope styles to the tag itself |
| `Unexpected 'STRING_START'` inside `[…]` styles | Quoted dynamic value: `[h:"{x}%"]` | Drop quotes: `[h:{x}%]` — values inside `[]` are CSS literals, expressions are wrapped in `{}` |
| Compile error on ternary inside `[…]` | `[bgc: a ? 'x' : 'y']` | Wrap the expression: `[bgc: {a ? 'x' : 'y'}]` |
| `SyntaxError` with no useful location | Mixed tabs and spaces in indentation | Use **tabs only**. One stray space breaks the lexer |
| Range parse error | `[1..24]` written without spaces | `[1 .. 24]` — spaces around `..` are required |
| Parse error around `=>` | JS arrow function | Replace with `do(x) ...` callback form |
| Parse error around `</tag>` | Explicit closing tag | Delete it. Imba tags close by indentation; there is no closing-tag syntax |
| Tag attribute won't parse | Compound boolean: `disabled=!email or !password` | Extract a getter: `get valid do email and password`, then `disabled=!valid` |
| `Unexpected token` after `#name` | Trying to write a comment without a space | Comments require a space: `# note`. Without the space, `#note` is a private-field reference |

## Runtime / behavioral

| Symptom | Cause | Fix |
|---|---|---|
| TS error: "Property X does not exist on type 'Window'" | Browser global called without `window.` | Prefix: `window.structuredClone(x)`, `window.fetch(...)` |
| Nothing renders, no error | Forgot to call `imba.mount` | Add `imba.mount <App>` at the entry point |
| Re-render not happening after mutation | Mutated nested object instead of replacing | Reassign at the reactive root: `list = [...list, item]`, or use `imba.commit!` |
| `@click.outside` never fires | Listener not in a `<global>` wrapper | Wrap the modal/menu in `<global>` |
| `@click.outside` fires immediately on open | The opening click bubbles to `outside` | Add `@click.stop` on the trigger button |
| Optional-chain on object doesn't compile | Used JS `?.` | Imba uses `..` for safe navigation: `user..name` |
| `style` getter does nothing / inline styles broken | Defined `get style` on a tag, shadowing built-in | Rename to `get theme` (or anything not named `style`) |
| Static asset 404 in dev | Raw string in `src=` | Move path into `const images = {…}` and reference `images.logo`. The bundler only resolves asset references through identifiers, not literals |
| `import` of a local file fails | Missing `.imba` extension | Add it: `import './a.imba'`. npm packages don't need it |
| `import` succeeds but symbol is `undefined` | Source file forgot `export` | Add `export` to the `def`/`class`/`const`/`tag` |
| Tag renders nothing or only on first paint | `def render` wrapping the template | **Delete the `def render` line and dedent its body one level.** Imba autorenders — the template must live at the top level of the tag body. See "Killing `def render`" below |

## Killing `def render`

Many migrants from React/Vue write `def render` wrapping the template. **Imba has no `render` method.** The template body sits directly under the `tag` declaration. Autorender re-runs it on observable change.

```imba
# ❌ wrong — template is hidden inside a method autorender doesn't call
tag counter
	count = 0
	def render
		<self>
			<button @click=(count++)> "Count: {count}"

# ✅ correct — template at top level of the tag body
tag counter
	count = 0
	<self>
		<button @click=(count++)> "Count: {count}"
```

When fixing a file, the mechanical edit is: delete the `def render` line and dedent every line of its body one tab to the left.

## Reactivity gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `@autorun` runs once and stops | Read happens before the autorun, value cached outside | Move the read inside the autorun body |
| `@computed` value stale | Underlying field is not reactive (e.g. plain object property mutated in place) | Reassign the whole field, or use `@observable` on the inner object |
| Form `bind=` doesn't update | Bound to a getter or computed property without a setter | Bind to a plain field, or define `set` accessor |

## When the table doesn't help

1. Read the actual file with the `Read` tool — most "weird" errors are an indentation or quoting issue you'll spot in the source.
2. Check `node_modules/imba/src/compiler/` in the user's project for the relevant lexer/parser file. The compiler is the source of truth when docs and behavior disagree.
3. Look at `references/syntax.md` for the canonical form of whatever construct is failing.
