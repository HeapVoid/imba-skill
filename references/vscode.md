# Imba + VS Code Reference

VS Code MCP, Playwright, TypeScript setup, file editing workflow.

---

## TypeScript Setup для Imba-проекта

### tsconfig.json (рабочий шаблон)

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["ESNext", "DOM"],
    "module": "Preserve",
    "moduleResolution": "Bundler",
    "noEmit": true,
    "skipLibCheck": true,
    "target": "ESNext",
    "typeRoots": ["./node_modules/@types", "./node_modules/imba/typings", "./types"]
  },
  "include": ["src/**/*", "types/**/*.d.ts"],
  "exclude": ["node_modules", "public"]
}
```

### Папка `types/` — кастомные фиксы типов

```ts
// types/imba.d.ts
interface HTMLElement {
  ease?: any;  // атрибут ease не входит в типизации Imba
}
```

Новые атрибуты/свойства которые TS не видит — добавлять сюда.

---

## Session Startup Checklist

### 1. VS Code MCP

Вызвать `get_diagnostics_code` на любом файле проекта. Если доступен — ок.

Если недоступен — попросить пользователя подключить: открыть панель расширения Claude Code в VS Code и вручную подключить сервер `vscode`. Не продолжать работу до подключения.

### 2. Playwright MCP + Live Preview

Вызвать `browser_snapshot`. Если доступен — ок.

Затем сделать `browser_navigate` на URL проекта (взять из CLAUDE.md проекта). После этого работать с текущей страницей до конца сессии без повторной навигации.

**Как устроен dev-сервер:**
- Статические файлы хостит **VS Code Live Preview** — расширение должно быть активно
- Live Preview запускается автоматически при открытии проекта
- Порт по умолчанию: **3000**, сервер раздаёт **корень проекта**
- Конкретный URL — в CLAUDE.md проекта (например `http://127.0.0.1:3000/public/index.html`)
- Если `ERR_CONNECTION_REFUSED` — Live Preview не запущен, попросить пользователя открыть его

**Компиляция происходит автоматически** через PostToolUse хук. `bun dev` запускать вручную не нужно.

**Конфиг `.mcp.json`:**
```json
"playwright": {
  "command": "npx",
  "args": ["@playwright/mcp@latest", "--headless"]
}
```
`--headless` обязателен. Опция `--url` не существует — не использовать.

---

## Проверка lint-ошибок

`get_diagnostics_code` — показывает ошибки TypeScript/линтера без запуска компилятора.

После правок не проверять автоматически — диагностика появляется с задержкой и даёт false negatives сразу после изменений. Пользователь сам попросит когда нужно.

---

## Visual Verification — Playwright

**Приоритет (быстрее → медленнее):**
1. `browser_snapshot` — HTML структура + element refs
2. `browser_console_messages` — runtime ошибки JS
3. `browser_evaluate` — computed styles, bounding rects, DOM properties
4. `browser_take_screenshot` — только когда внешний вид нельзя понять из выше

Пользователь сам скажет когда нужна проверка браузером. **Не делать скриншоты проактивно.**

---

## File Editing — приоритет инструментов

### Порядок приоритетов

1. **Встроенные инструменты Claude Code** (`Edit`, `Write`, `Read`, `Glob`, `Grep`) — использовать по умолчанию
2. **VS Code MCP** (`replace_lines_code`, `create_file_code`, `move_file_code` и др.) — если встроенные не справляются
3. **Bash / ОС** — только если ни встроенные, ни MCP не подходят

**Почему избегать Bash для работы с файлами:**
- VS Code не видит изменений — они не отображаются в git diff в редакторе
- Каждое действие требует отдельного подтверждения от пользователя

### Когда использовать VS Code MCP вместо встроенных

- `move_file_code` / `rename_file_code` — переместить/переименовать **с автообновлением всех импортов**
- `get_diagnostics_code` — lint/TypeScript ошибки без запуска компилятора
- `get_document_symbols_code`, `get_symbol_definition_code`, `search_symbols_code` — навигация по символам
- `replace_lines_code` — если `Edit` не справился (конфликт при точном матчинге)

**Ограничение VS Code MCP:** работает только с файлами внутри воркспейса. Файлы вне воркспейса (`~/.skills/`, `~/.claude/` и т.д.) — только через `Write`/`Edit`.

| Задача | Инструмент |
|--------|-----------|
| Небольшая правка | `Edit` |
| Новый файл / полная перезапись | `Write` |
| Читать файл | `Read` |
| Поиск файлов | `Glob` |
| Поиск по содержимому | `Grep` |
| Переместить/переименовать (+ импорты) | `move_file_code` / `rename_file_code` |
| Проверить ошибки | `get_diagnostics_code` |
| Найти символ | `search_symbols_code` |
