# Conclave — Многораундовые дебаты нейросетей

Утилита для запуска структурированной дискуссии между Gemini, GPT и Claude.  
Каждый раунд участники читают ответы оппонентов и продолжают спор.  
В конце Claude подводит итог и выносит вердикт.

## Установка

```bash
cp conclave ~/bin/conclave
chmod +x ~/bin/conclave
```

Зависимости: те же что у `council` — [gemini-cli](https://github.com/google-gemini/gemini-cli) и [codex](https://github.com/openai/codex).

## Использование

```bash
echo "вопрос или тема" | conclave [--rounds N] [роль]
```

Параметры:
- `--rounds N` — количество раундов (по умолчанию `3`)
- роль — `review`, `alt`, `devil`, `sql`, `sec`, или произвольный текст

Примеры:
```bash
echo "стоит ли использовать ORM в production?" | conclave

echo "SELECT * FROM orders WHERE status = 'new'" | conclave --rounds 4 sql

echo "архитектура микросервисов vs монолит" | conclave --rounds 2 alt
```

## Как работает

**Раунд 1** — Gemini и GPT отвечают независимо на задачу.

**Раунды 2–N** — каждый участник получает полный транскрипт предыдущих раундов и продолжает дебаты: оспаривает аргументы, указывает на фактические ошибки, развивает свою позицию.

**Финал** — маркер `=== ЗАДАНИЕ ДЛЯ CLAUDE ===` для итогового вердикта.

Участники общаются в формате живых дебатов, не академического доклада. Если позиция фактически неверна — оппоненты обязаны на это указать.

## Интеграция с Claude

Если ты запускаешь conclave из Claude Code (через CLAUDE.md), Claude участвует в дебатах через свою начальную позицию. Формат входных данных:

```
ЗАДАЧА: [вопрос или тема]

МОЙ ОТВЕТ (Claude): [анализ и позиция Claude]
```

Gemini и GPT видят позицию Claude в первом раунде и реагируют на неё.  
Claude читает транскрипт после завершения и выносит финальный вердикт.

## Роли

| Роль | Описание |
|------|----------|
| `review` | Code review: баги, edge cases, production-риски |
| `alt` | Альтернативный подход и trade-offs |
| `devil` | Devil's advocate: что может сломаться |
| `sql` | DBA-анализ: deadlocks, N+1, locking |
| `sec` | Security review: injection, auth bypass, SSRF |
| `"текст"` | Произвольная роль |

## Логи

Каждая сессия пишется в `~/concilium/sessions/conclave_TIMESTAMP.md`.  
Папка `sessions/` в `.gitignore`.

## Запуск в фоне

```bash
echo "тема" | conclave --rounds 4 > /tmp/conclave.txt 2>&1 &
tail -f /tmp/conclave.txt
```
