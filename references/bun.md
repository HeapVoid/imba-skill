# Imba + Bun Reference

Bun toolkit and bimba-cli compiler workflow for Imba projects.

---

## Bun — основы

All-in-one toolkit: runtime + package manager + test runner + bundler. TypeScript и JSX из коробки.

### Основные команды

```bash
bun install              # установить зависимости
bun add <pkg>            # добавить пакет (--dev/-d для devDependencies)
bun remove <pkg>         # удалить пакет
bun run <script>         # запустить script из package.json
bun run <file.ts>        # запустить файл напрямую
bunx <pkg>               # запустить бинарник пакета (= npx, но быстрее)
bun test                 # запустить тесты
bun build                # бандлер
```

`bunx` сначала ищет в локальном `node_modules`. Флаг `--bun` форсирует Bun-рантайм (ставить ДО имени пакета): `bunx --bun vite`

---

## bimba-cli — компилятор Imba

Репозиторий: https://github.com/HeapVoid/bimba

### Три режима работы

**Dev server с HMR — frontend разработка:**
```bash
bunx bimba src/index.imba --serve --port 5200 --html public/index.html
```

**Bundle — production сборка:**
```bash
bunx bimba <entry.imba> --outdir <folder> [flags]
```

**Backend (рантайм через Bun) — plugin:**
```toml
# bunfig.toml
preload = ["bimba-cli/plugin.js"]
```
```bash
bun run src/index.imba
bun --watch run src/index.imba
```

---

### Dev server (`--serve`)

HMR сервер для Imba. Компилирует `.imba` on demand, без бандлинга.

**Как работает:**
1. Читает HTML-файл (`--html`), удаляет из него существующий importmap
2. Строит importmap из `package.json` — `.imba` пакеты → `/node_modules/...`, остальные → `esm.sh`
3. Заменяет `<script data-entrypoint>` на `<script src="/src/index.imba">` и вставляет importmap выше
4. Инжектирует HMR-клиент: перехватывает `customElements.define`, при изменении файла делает прототипный своп + сбрасывает `innerHTML` затронутых элементов

**HTML setup** — добавить атрибут `data-entrypoint` на тег скрипта с бандлом:
```html
<script type='module' src="./js/index.js" data-entrypoint></script>
```
В production он грузит `./js/index.js`. В dev-сервере заменяется на `/src/index.imba`.

**Статика:** файлы разрешаются сначала относительно директории HTML-файла, затем из корня проекта (для `node_modules`, `src` и т.д.).

**WebSocket HMR:** при изменении `.imba` файла клиент делает `import('/src/file.imba?t=...')`, находит все инстансы обновлённых классов через `instanceof` (включая подклассы), сбрасывает их `innerHTML` и вызывает `imba.commit()`.

---

### Все CLI-флаги

| Флаг | Default | Описание |
|------|---------|----------|
| `--outdir <path>` | — | **Обязателен** для bundle. Папка для JS-вывода |
| `--watch` | false | Следить за изменениями, пересобирать |
| `--clearcache` | false | Удалить `.cache/` при выходе (Ctrl+C) |
| `--minify` | **true** | Минификация (включена по умолчанию!) |
| `--sourcemap <inline\|external\|none>` | none | Source maps |
| `--target <browser\|node>` | browser | Платформа (`node` не работает в Bun) |
| `--serve` | false | Dev server с HMR вместо бандлинга |
| `--port <number>` | 5200 | Порт dev сервера |
| `--html <path>` | auto | HTML-файл (авто: `index.html`, `public/index.html`, `src/index.html`) |

### Как работает bundle под капотом

1. Вызывает `Bun.build()` с плагином `imba-plugin`
2. Плагин перехватывает все импорты `*.imba`
3. Для каждого файла: вычисляет хеш пути + mtime → ищет в `.cache/`
4. Если кеш актуален — отдаёт из кеша без перекомпиляции
5. Если нет — компилирует через `imba/compiler`, кладёт в `.cache/`

**Watch-режим:** следит за **всей директорией** entrypoint рекурсивно. Entrypoint лучше держать в `src/`, а не в корне.

**Кеш:** `.cache/` в CWD проекта. Инвалидируется по mtime файла.

### Вывод bimba при компиляции

```
src/api.enrich.pb.imba - compiled    # ✅ Файл успешно скомпилирован
                                     # (результат сохранён в .cache/)
```

Когда bimba пишет **"compiled"** — файл только что скомпилирован. Всё ок.

Когда **вообще нет вывода** — файл взят из `.cache/` без перекомпиляции (лог для хита кэша закомментирован в плагине `bimba-cli/plugin.js`).

### Типичные команды проекта

```bash
# Dev server с HMR
bunx bimba src/index.imba --serve --port 5200 --html public/index.html

# Production сборка
bunx bimba src/index.imba --outdir public/js --minify

# Dev watch (без HMR-сервера)
bunx bimba src/index.imba --outdir public/js --watch --clearcache

# Проверить компиляционные ошибки
bunx bimba src/index.imba --outdir public/js 2>&1 | head -50
```

Точные пути смотреть в `package.json` проекта (секция `scripts`).

---

## Три уровня ошибок

| Уровень | Инструмент |
|---------|------------|
| **Lint / TypeScript** | `get_diagnostics_code` (VS Code MCP) |
| **Компиляция** | `bunx bimba ... 2>&1 \| head -50` |
| **Runtime** | `browser_console_messages` (Playwright) |

Проверять последовательно. Компиляционные ошибки выводятся в stderr — `2>&1` обязателен.

---

## Managing Background Processes

```bash
ps aux | grep -E 'bimba|bun' | grep -v grep
pkill -f bimba
```

---

## Testing

```bash
bun test
bun test --watch
```

Тест-файлы: `.test.js` / `.test.ts` в папке `tests/`. Jest-совместимый API.

---

## Автоматическая компиляция через хук

В проектах настроен **PostToolUse хук** в `~/.claude/settings.json`: при каждом сохранении `.imba` файла через `Edit` или `Write` автоматически запускается bimba.

**Что это значит:**
- Написал и сохранил `.imba` файл → компиляция запускается автоматически
- **Не нужно** запускать `bun dev` вручную
- **Не нужно** держать watch-процесс в фоне
- Если нужно явно проверить ошибки компиляции:

```bash
bunx bimba src/index.imba --outdir public/js 2>&1 | head -50
```
