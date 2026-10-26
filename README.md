Репозиторий перенесён и заархивирован. Актуальная версия: https://github.com/sonix-zhx/koderlinetech-ai-cicd-1c

# ИИ в CI/CD для 1С: демо

Демо к докладу «Как встроить ИИ в CI/CD в 1С» (Koderline Tech, 27.10.2026).
Мини-конфигурация 1С в `src/` (выгрузка EDT, упрощённая) и четыре сценария GitHub Actions с агентом OpenCode.

## Сценарии

| Workflow | Триггер | Что делает |
|---|---|---|
| `ai-review.yml` | `pull_request` | Диф по `.bsl` уходит агенту. Ответ комментарием в PR. Последняя строка `VERDICT: PASS` или `FAIL`, при `FAIL` check красный. Строгость задаёт промпт |
| `ai-sonar.yml` | `pull_request` | BSL Language Server строит отчёт, `jq` оставляет замечания по изменённым файлам, агент группирует и предлагает исправления. Только комментарий, без статуса |
| `ai-release-notes.yml` | push тега `v*` | Агент сам читает коммиты между тегами (`git log`, `git show` для неинформативных). Результат: GitHub Release и коммит в `CHANGELOG.md` |
| `ai-command.yml` | комментарий `/opencode ...` или `/oc ...` в PR | Готовая интеграция OpenCode правит код и коммитит в ветку PR |

Промпты: `.github/prompts/`. Правила для агента: `AGENTS.md`.

В `ai-sonar.yml` отчёт строится на лету. С живым SonarQube его берут через API
(`/api/issues/search?pullRequest=<номер>`), остальные шаги те же.

## Настройка

1. Ключ OpenCode Zen: Settings, Secrets and variables, Actions, секрет `OPENCODE_API_KEY`.
   Ключ создаётся в консоли OpenCode Zen. Имя переменной `OPENCODE_API_KEY` взято из документации Zen.
2. Модель задана в одном месте: `opencode.json`, поле `model` (сейчас `opencode/deepseek-v4.1-flash-free`).
   Бесплатные модели Zen действуют ограниченное время, при смене правится только эта строка.
3. Интеграция для `/opencode`:
   1. Установить GitHub App: https://github.com/apps/opencode-agent (в этот репозиторий).
   2. Секрет из п. 1 уже подходит. Workflow `ai-command.yml` в репозитории готов.
   3. Альтернатива без App: `opencode github install` в клоне репозитория сделает шаги за вас.
4. Список тех, кто кроме автора PR может давать команды: `OPENCODE_OWNERS` в `ai-command.yml` (по умолчанию `sonix-zhx`).

## Повторное ревью после правок агента

Пуш токеном `GITHUB_TOKEN` не запускает другие workflow, это правило GitHub.
Интеграция OpenCode по умолчанию работает через токен GitHub App (обмен OIDC), поэтому
коммит агента запускает `ai-review.yml` повторно. Это вывод из правил GitHub, в документации
OpenCode прямо не сказано: проверить при первом прогоне.
Режим `use_github_token: true` пушит от `GITHUB_TOKEN`, повторного ревью не будет.
Для него нужен PAT или токен другого App: передать и как `GITHUB_TOKEN`, и как `token` в `actions/checkout`.

## Демо-ветки

PR открывать в `main`. Тег `v1.0.0` стоит на начальном состоянии.

| Ветка | Ожидание |
|---|---|
| `demo/review-bad` | запрос в цикле, пустое `Исключение`, пара рекомендаций: `VERDICT: FAIL` |
| `demo/review-fixed` | та же функциональность исправлена: `VERDICT: PASS` |
| `demo/sonar` | много замечаний BSL LS разного уровня, комментарий без статуса |

Для `ai-release-notes.yml` после демо поставить тег `v1.1.0` на `main`.

## Проверить при первом прогоне

- Имя jar в релизе BSL Language Server (`*exec.jar`) и имя файла отчёта `bsl-json.json`.
- Что `opencode run "<текст>" -f файл` берёт текст и файл как задумано, и вид вывода (шапка отсекается скриптом `clean-output.sh`).
- Что модель `opencode/deepseek-v4.1-flash-free` ещё бесплатна и ключ `OPENCODE_API_KEY` принимается.
- Что `OPENCODE_PERMISSION` разрешает агенту `git log` и `git show` в релизе.
- Что `main` не защищён от пуша `github-actions[bot]` (коммит в `CHANGELOG.md`).
