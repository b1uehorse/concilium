# задача

собери CLI-утилиту `council` — двусторонний мост между Claude Code и Gemini CLI для "консилиума нейросетей".

## что это делает

1. читает clipboard (ответ из CC)
2. оборачивает в системный промпт (роль: review / alternative / devil's advocate / sql / security / произвольный)
3. отправляет в `gemini -p` (headless mode, без интерактива)
4. получает ответ gemini
5. форматирует и кладёт обратно в clipboard — готово для вставки в CC
6. ведёт markdown-лог всех раундов в `~/.council/session.md`

## технические требования

- язык: bash. один файл, без внешних зависимостей кроме gemini cli
- clipboard: `pbpaste`/`pbcopy` на macOS, `wl-paste`/`wl-copy` на wayland (Linux/Hyprland). определять по `uname`
- gemini cli вызывается как `gemini -p "промпт"` — это headless mode, вход через stdin тоже работает: `echo "текст" | gemini`
- gemini cli авторизуется через Google аккаунт, API ключ НЕ нужен
- лог сессии: `~/.council/session.md`, каждый раунд — h2 заголовок с timestamp и source (CC/Gemini)
- трекинг раундов: файл `~/.council/.round`, инкрементится при каждом вызове

## команды

```
council                     # review: clipboard → gemini → clipboard
council alt                 # альтернативный подход
council devil               # devil's advocate (найди почему сломается)
council sql                 # DBA review (deadlocks, N+1, locking)
council sec                 # security review
council "произвольный текст" # кастомная роль
council loop 3              # 3 раунда: CC→Gemini→CC→Gemini→CC→Gemini, с паузами между раундами для ручной вставки в CC
council log                 # показать лог сессии
council clear               # очистить лог и сбросить раунды
council help                # справка
```

## системные промпты по ролям

- **review**: "ты второй инженер на code review. проанализируй решение коллеги. укажи баги, слабые места, edge cases, потенциальные проблемы в production. если всё ок — скажи коротко что ок и почему. будь конкретен, без воды."
- **alt**: "предложи альтернативный подход к этой задаче. не критикуй исходное решение — покажи другой путь, объясни trade-offs."
- **devil**: "играй роль devil's advocate. найди все причины почему это решение может сломаться в production. подумай про edge cases, race conditions, масштабирование, человеческий фактор."
- **sql**: "ты senior DBA. проверь SQL/миграции/запросы на: deadlock потенциал, N+1, отсутствие индексов, implicit conversions, locking issues, data loss risks. будь конкретен."
- **sec**: "проведи security review. ищи: injection, auth bypass, secrets в коде, SSRF, path traversal, privilege escalation. будь конкретен."
- если аргумент не совпадает ни с одной ролью — использовать его как произвольный системный промпт

## формат ответа для clipboard

после получения ответа от gemini, в clipboard кладётся:

```
[консилиум round N — роль]

второй инженер (Gemini) говорит:

{ответ gemini}
```

это готово для вставки в CC.

## формат лога

```markdown
## round 1 — CC (14:32:05)

{текст из clipboard}

---

## round 1 — Gemini (14:32:18)

{ответ gemini}

---
```

## loop режим

`council loop N [роль]` запускает N раундов:
1. читает clipboard (ответ CC)
2. отправляет gemini
3. показывает ответ + копирует в clipboard
4. ждёт Enter (пользователь вставляет в CC, копирует новый ответ CC)
5. повторяет

## план реализации

1. создай `~/bin/council` (mkdir -p ~/bin если не существует)
2. напиши скрипт по спецификации выше
3. `chmod +x ~/bin/council`
4. проверь что `~/bin` в PATH (если нет — выведи инструкцию)
5. протестируй:
   - `council help` — должна вывести справку
   - `council log` — должна отработать без ошибок (пустой лог)
   - `council clear` — должна отработать
   - положи тестовый текст в clipboard (`echo "SELECT * FROM users WHERE id = 1" | pbcopy` или `wl-copy`), запусти `council sql` — проверь что gemini отвечает и ответ попадает в clipboard
   - проверь что лог пишется в `~/.council/session.md`
6. создай `~/bin/council-readme.md` с документацией

## ограничения

- gemini cli должен быть установлен и авторизован. если `which gemini` не находит — выведи ошибку с инструкцией: `npm install -g @google/gemini-cli && gemini` (первый запуск попросит авторизацию)
- на этапе тестирования gemini может быть недоступен — это ок, тестируй структуру скрипта (парсинг аргументов, clipboard, логирование), а сетевые вызовы пометь как "requires manual test"
- не используй jq и не используй python — только bash и стандартные утилиты

## стиль кода

- shellcheck-clean
- комментарии на русском
- переменные UPPER_CASE для конфигурации, lower_case для локальных
- set -euo pipefail
- функции для каждого блока логики

## автоматический консилиум из CC

когда пользователь просит "спроси gemini", "консилиум", "второе мнение" или подобное — вызови council через bash:

```bash
echo "текст для анализа" | council [роль]
```

council выведет ответ gemini в stdout. прочитай его и учти в дальнейшей работе.

доступные роли: review, alt, devil, sql, sec, или произвольный текст в кавычках.

пример:
```bash
echo "SELECT * FROM users WHERE deleted = 0" | council sql
```
