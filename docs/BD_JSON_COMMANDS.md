# Справочник JSON-ответов команд `bd`

Этот документ описывает, что возвращают ключевые команды `bd --json`, и предлагает примеры Python-обёрток для их вызова. Поля объектов соответствуют структурам данных `Issue` и `Dependency` из исходников CLI, поэтому приведённые примеры можно использовать для сериализации/десериализации.

## Покрытие команд

Ниже перечислены все команды, зарегистрированные в `cmd/bd` через `rootCmd.AddCommand` (и их подкоманды), которые либо уже описаны в разделах ниже, либо отмечены как текстовые утилиты без стабильного JSON:

- Задачи: `ready`, `blocked`, `stats`, `stale`, `list`, `show`, `update`, `edit`, `close`, `reopen`, `create`, `delete`, `deleted`, `restore`, `count`.
- Комментарии и метки: `comments add/list` (включая alias `comment`), `label add/remove/list/list-all`, `search` (с фильтрами), `update`, `show`.
- Зависимости и эпики: `dep add/remove/list/tree/cycles`, `repair-deps`, `duplicates`/`merge`, `epic status`, `epic close-eligible`.
- Импорт/экспорт и миграции: `export`, `import`, `migrate --inspect`, `migrate hash-ids`, `migrate tombstones`, `migrate issues`, `migrate-sync`, `rename-prefix`, `validate`, `detect-pollution`.
- Обслуживание БД/демона: `daemon`, `daemons list/health/logs/stop/restart/killall`, `sync`, `hooks install/uninstall/list`, `config get/set/unset/list`, `prime`, `info`, `status`, `doctor`, `compact`, `cleanup`, `clean` (текстовый), `upgrade status/review/ack`, `version`.
- Репозитории и интеграции: `repo add/remove/list/sync`, `migrate-sync`, `setup claude|cursor|aider`, `onboard`, `quickstart`, `workflow`, `init` (включая `--team`, `--contributor`), `message send/inbox/read/ack`, `jira sync/status`, `template list/show/create`, `tips`.

Если появится новая команда в `cmd/bd/*.go`, добавьте её в этот список и в соответствующий раздел ниже (или в блок «Команды без `--json`»), чтобы сохранить полноту справочника. Аудит по `rg "rootCmd.AddCommand"` для текущего коммита подтверждает, что все зарегистрированные команды уже перечислены и описаны в этом файле.

## Базовые сведения о структуре данных

### Объект `Issue`

Основные поля, которые появляются в ответах команд поиска и чтения:

```json
{
  "id": "bd-42",
  "title": "Fix authentication bug",
  "description": "…",
  "design": "…",                    // опционально
  "acceptance_criteria": "…",       // опционально
  "notes": "…",                     // опционально
  "status": "open|in_progress|blocked|closed|tombstone",
  "priority": 0,
  "issue_type": "bug|feature|task|epic|chore",
  "assignee": "alice",              // опционально
  "estimated_minutes": 90,           // опционально
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-02T08:00:00Z",
  "closed_at": "2024-01-03T18:00:00Z",  // только для закрытых задач
  "close_reason": "Completed",      // опционально
  "external_ref": "gh-123",         // опционально
  "compaction_level": 0,
  "compacted_at": null,
  "compacted_at_commit": null,
  "original_size": 0,
  "labels": ["bug", "critical"],
  "dependencies": [
    {"issue_id": "bd-42", "depends_on_id": "bd-41", "type": "blocks"}
  ],
  "comments": [
    {"issue_id": "bd-42", "author": "alice", "body": "…", "created_at": "…"}
  ],
  "deleted_at": null,
  "deleted_by": "",
  "delete_reason": "",
  "original_type": ""
}
```

### Объект `Dependency`

```json
{"issue_id": "bd-42", "depends_on_id": "bd-41", "type": "blocks|related|parent-child|discovered-from"}
```

## Команды обзора и статуса

### `bd info`

- **Python-обёртка:** `def bd_info(schema: bool = False, whats_new: bool = False) -> dict`
- **JSON-ответ:** сводка окружения и демона. Пример для `--json`:

```json
{
  "database_path": "/workspace/.beads/beads.db",
  "mode": "daemon|direct",
  "daemon_connected": true,
  "socket_path": "/tmp/beads.sock",
  "daemon_version": "0.21.0",
  "daemon_status": "healthy",
  "daemon_compatible": true,
  "daemon_uptime": "1h2m",
  "issue_count": 128,
  "config": {"issue_prefix": "bd"},
  "schema": {
    "tables": ["issues", "dependencies", "labels", "config", "metadata"],
    "schema_version": "…",
    "config": {"issue_prefix": "bd"},
    "sample_issue_ids": ["bd-1", "bd-2"],
    "detected_prefix": "bd"
  }
}
```

### `bd status`

- **Python-обёртка:** `def bd_status() -> dict`
- **JSON-ответ:** состояние демона и счётчики. Типичный фрагмент:

```json
{
  "mode": "daemon|direct",
  "daemon": {
    "connected": true,
    "socket": "/tmp/beads.sock",
    "health": "healthy",
    "version": "0.21.0"
  },
  "statistics": {"total_issues": 128, "open": 90, "closed": 38}
}
```

## Поиск и выборка задач

### `bd ready`

- **Python-обёртка:** `def bd_ready(limit: int | None = None, days: int | None = None) -> list[dict]`
- **JSON-ответ:** список объектов `Issue` без блокеров. Пример: `[ {…Issue…}, {…} ]`.

### `bd blocked`

- **Python-обёртка:** `def bd_blocked() -> list[dict]`
- **JSON-ответ:** массив `BlockedIssue`, который объединяет `Issue` с полями `blocked_by_count` и `blocked_by` (массив ID блокирующих задач).

### `bd stats`

- **Python-обёртка:** `def bd_stats() -> dict`
- **JSON-ответ:** агрегаты `Statistics` (total/open/in_progress/closed/blocked/ready/tombstone/epics_eligible_for_closure` и `average_lead_time_hours`).

### `bd stale`

- **Python-обёртка:** `def bd_stale(days: int = 30, status: str | None = None, limit: int | None = None) -> list[dict]`
- **JSON-ответ:** массив `Issue` с указанной давностью обновления.

### `bd list` / `bd search`

- **Python-обёртка:** `def bd_list(**filters) -> list[dict]`
  - Примеры аргументов: `status`, `priority`, `type`, `assignee`, `label`, `label_any`, `title_contains`, `desc_contains`, диапазоны дат.
- **JSON-ответ:** массив `Issue` по заданным фильтрам.

### `bd show`

- **Python-обёртка:** `def bd_show(ids: list[str]) -> list[dict]`
- **JSON-ответ:** детальные `Issue` для каждой переданной ID (включая зависимости, метки и комментарии).

### `bd count`

- **Python-обёртка:** `def bd_count(*, status: str | None = None, issue_type: str | None = None, priority: int | None = None, assignee: str | None = None, labels: list[str] | None = None, labels_any: list[str] | None = None, title_contains: str = "", desc_contains: str = "", notes_contains: str = "", group_by: str | None = None, ids: list[str] | None = None, created_after: str | None = None, created_before: str | None = None, updated_after: str | None = None, updated_before: str | None = None) -> dict`
- **JSON-ответ:** либо простое `{ "count": 42 }`, либо с группировкой `{ "total": 42, "groups": [ { "group": "p1", "count": 10 }, … ] }` по полям `priority|type|assignee|label`.

## Создание и изменение задач

### `bd create`

- **Python-обёртка:**
  ```python
  def bd_create(
      title: str,
      *,
      description: str = "",
      issue_type: str = "task",
      priority: int = 2,
      assignee: str | None = None,
      labels: list[str] | None = None,
      deps: list[str] | None = None,
      parent: str | None = None,
      estimate: int | None = None,
      external_ref: str | None = None,
      explicit_id: str | None = None,
      repo: str | None = None,
  ) -> dict
  ```
- **JSON-ответ:** созданный объект `Issue` (включая присвоенную ID и добавленные зависимости).

### `bd update`

- **Python-обёртка:**
  ```python
  def bd_update(
      ids: list[str],
      *,
      status: str | None = None,
      priority: int | None = None,
      assignee: str | None = None,
      description: str | None = None,
      design: str | None = None,
      notes: str | None = None,
      acceptance: str | None = None,
      estimate: int | None = None,
      external_ref: str | None = None,
      labels: list[str] | None = None,
      deps: list[str] | None = None,
  ) -> list[dict]
  ```
- **JSON-ответ:** массив обновлённых `Issue`.

### `bd edit`

- **Python-обёртка:** `def bd_edit(id: str, *, field: str = "description") -> dict`
- **JSON-ответ:** обновлённая задача `Issue` после редактирования указанного поля (title/description/design/notes/acceptance_cri
teria). Флаг `--json` возвращает объект задачи даже при интерактивном редактировании через `$EDITOR`.

### `bd close` / `bd reopen`

- **Python-обёртка:**
  - `def bd_close(ids: list[str], *, reason: str = "") -> list[dict]`
  - `def bd_reopen(ids: list[str], *, reason: str = "") -> list[dict]`
- **JSON-ответ:** массив `Issue` в новом статусе (closed или open/in_progress).

### `bd delete`

- **Python-обёртка:** `def bd_delete(ids: list[str], *, reason: str = "") -> list[dict]`
- **JSON-ответ:** массив помеченных как tombstone задач (`status: "tombstone"`, заполнены `deleted_at`, `deleted_by`, `delete_reason`).

## Метки, зависимости и комментарии

### `bd dep add/remove/list`

- **Python-обёртки:**
  - `def bd_dep_add(issue_id: str, depends_on_id: str, dep_type: str = "blocks") -> dict`
  - `def bd_dep_remove(issue_id: str, depends_on_id: str, dep_type: str = "blocks") -> dict`
  - `def bd_dep_list(issue_id: str) -> list[dict]`
- **JSON-ответ:** добавление/удаление возвращает подтверждение `{ "issue_id": "…", "depends_on_id": "…", "type": "…" }`; list — массив зависимостей для задачи.

### `bd dep tree`

- **Python-обёртка:**
  - `def bd_dep_tree(issue_id: str, *, direction: str = "down", status: str | None = None, max_depth: int = 10, show_all_paths: bool = False) -> list[dict]`
  - Используйте `direction="up|down|both"` для зависимых/зависимостей/обоих направлений; `max_depth` ограничивает глубину.
- **JSON-ответ:** массив узлов дерева `TreeNode`, каждый комбинирует `Issue` с полями глубины и связей:
  `{ "id": "bd-42", "title": "…", "depth": 0, "parent_id": "bd-1", "truncated": false, …Issue… }`.

### `bd dep cycles`

- **Python-обёртка:** `def bd_dep_cycles() -> list[list[dict]]`
- **JSON-ответ:** массив циклов; каждый цикл — список объектов `Issue`, участвующих в обнаруженной циклической зависимости.

### `bd label add/remove/list/list-all`

- **Python-обёртки:**
  - `def bd_label_add(ids: list[str], label: str) -> list[dict]`
  - `def bd_label_remove(ids: list[str], label: str) -> list[dict]`
  - `def bd_label_list(id: str) -> list[str]`
  - `def bd_label_list_all() -> list[str]`
- **JSON-ответ:**
  - add/remove: массив объектов `{ "id": "bd-42", "labels": ["bug", "critical"] }`
  - list: массив строк
  - list-all: массив всех меток в базе.

### `bd comments add/list`

- **Python-обёртки:**
  - `def bd_comment_add(issue_id: str, body: str) -> dict`
  - `def bd_comment_list(issue_id: str) -> list[dict]`
- **JSON-ответ:** объект добавленного комментария либо массив комментариев `{ "issue_id": "bd-42", "author": "alice", "body": "…", "created_at": "…" }`.

## Эпики

### `bd epic status`

- **Python-обёртка:** `def bd_epic_status(eligible_only: bool = False) -> list[dict]`
- **JSON-ответ:** массив `EpicStatus`: `{ "epic": {…Issue…}, "total_children": 5, "closed_children": 3, "eligible_for_close": false }`.

### `bd epic close-eligible`

- **Python-обёртка:** `def bd_epic_close_eligible(dry_run: bool = False) -> list[dict]`
- **JSON-ответ:** при `--dry-run` возвращает тот же массив `EpicStatus`, показывая эпики, готовые к закрытию; без `--dry-run` возвращает итоговый список закрытых эпиков.

### `bd deleted`

- **Python-обёртка:** `def bd_deleted(include_children: bool = False) -> list[dict]`
- **JSON-ответ:** массив задач в статусе tombstone (аналогично `Issue`, плюс заполненные поля удаления).

### `bd repair-deps`

- **Python-обёртка:** `def bd_repair_deps(ids: list[str] | None = None, dry_run: bool = False) -> dict`
- **JSON-ответ:** отчёт об исправлениях зависимостей `{ "checked": n, "fixed": m, "dry_run": bool }`.

## Утилиты качества базы

### `bd duplicates` / `bd merge`

- **Python-обёртки:**
  - `def bd_duplicates(auto_merge: bool = False, dry_run: bool = False) -> dict`
  - `def bd_merge(source_ids: list[str], target_id: str, dry_run: bool = False) -> dict`
- **JSON-ответ:**
  - duplicates: группы дублей `{ "groups": [ ["bd-1", "bd-2"], … ], "auto_merged": [ … ] }`
  - merge: итоговая задача и перечень перенесённых полей `{ "target": {…Issue…}, "merged": ["bd-2", "bd-3"] }`

### `bd cleanup`

- **Python-обёртка:** `def bd_cleanup(older_than: int | None = None, cascade: bool = False, dry_run: bool = False) -> dict`
- **JSON-ответ:** счётчики удалённых записей `{ "deleted": 10, "cascaded": 2, "dry_run": true }`.

### `bd compact`

- **Python-обёртки:**
  - `def bd_compact_analyze(tier: int | None = None, limit: int | None = None) -> dict`
  - `def bd_compact_apply(issue_id: str, summary_path: str | None = None, summary_stdin: bool = False) -> dict`
  - `def bd_compact_stats() -> dict`
- **JSON-ответ:**
  - analyze: кандидаты `{ "candidates": [ {"id": "bd-42", "tier": 1, "size": 1024}, … ] }`
  - apply: обновлённый `Issue` (с заполненными `compaction_level`, `compacted_at`, `original_size`)
  - stats: агрегаты `{ "compacted": 5, "pending": 10, "savings_bytes": 12345 }`.

### `bd validate`

- **Python-обёртка:** `def bd_validate(fix: bool = False, dry_run: bool = False) -> dict`
- **JSON-ответ:** список проверок с полями `{ "name": "…", "status": "ok|warn|error", "fix_applied": bool, "details": "…" }` и сводным флагом `overall_ok`.

## Импорт, экспорт и миграции

### `bd export` / `bd import`

- **Python-обёртки:**
  - `def bd_export(path: str, *, dedupe_after: bool = False) -> dict`
  - `def bd_import(path: str, *, dry_run: bool = False, dedupe_after: bool = False, orphan_handling: str = "allow") -> dict`
- **JSON-ответ:**
  - export: `{ "path": "./.beads/issues.jsonl", "written": 200 }`
  - import: `{ "created": 5, "updated": 10, "skipped": 0, "orphan_handling": "allow", "dry_run": false }`

### `bd migrate --inspect`

- **Python-обёртка:** `def bd_migrate_inspect() -> dict`
- **JSON-ответ:** план миграции `{ "pending": ["bd-v20"], "warnings": [], "invariants_to_check": ["required_config_present"] }`.

### `bd migrate hash-ids` / `bd migrate tombstones` / `bd migrate issues`

- **Python-обёртки:**
  - `def bd_migrate_hash_ids(dry_run: bool = False) -> dict`
  - `def bd_migrate_tombstones(limit: int | None = None, dry_run: bool = False) -> dict`
  - `def bd_migrate_issues(path: str, dry_run: bool = False) -> dict`
- **JSON-ответ:** отчёты о миграции (количество затронутых записей, `dry_run`, путь или список перенесённых ID).

### `bd markdown import`

- **Python-обёртка:** `def bd_markdown_import(path: str, assignee: str | None = None, labels: list[str] | None = None) -> list[dict]`
- **JSON-ответ:** массив созданных задач по Markdown-файлу (каждый элемент — `Issue`).

## Сервисные команды

### `bd daemons list/health/logs/stop/restart/killall`

- **Python-обёртки:**
  - `def bd_daemons_list() -> list[dict]`
  - `def bd_daemons_health() -> list[dict]`
  - `def bd_daemons_logs(pid_or_path: str) -> dict`
  - `def bd_daemons_stop(pid_or_path: str) -> dict`
  - `def bd_daemons_restart(pid_or_path: str) -> dict`
  - `def bd_daemons_killall(force: bool = False) -> dict`
- **JSON-ответ:**
  - list/health: массив объектов `{ "workspace": "/workspace/beads", "pid": 12345, "status": "healthy", "version": "0.21.0" }`
  - logs: `{ "pid": 12345, "workspace": "…", "logs": "…stdout…" }`
  - stop/restart: `{ "stopped": true, "target": "12345" }` либо `{ "restarted": true, … }`
  - killall: `{ "killed": ["/workspace/beads"], "force": false }`

### `bd daemon`

- **Python-обёртка:** `def bd_daemon(no_auto_start: bool = False) -> dict`
- **JSON-ответ:** подтверждение запуска/остановки демона `{ "started": bool, "socket_path": "…", "pid": 12345 }` либо сообщение об ошибке.

### `bd sync`

- **Python-обёртка:** `def bd_sync() -> dict`
- **JSON-ответ:** подтверждение этапов синхронизации `{ "exported": true, "imported": true, "committed": true, "pushed": true }`.

### `bd hooks`

- **Python-обёртки:**
  - `def bd_hooks_install(force: bool = False, shared: bool = False) -> dict`
  - `def bd_hooks_uninstall() -> dict`
  - `def bd_hooks_list() -> dict`
- **JSON-ответ:** статусы установки/удаления (`{ "success": true, "message": "…", "shared": bool }`) или текущее состояние (`{ "hooks": [ { "name": "pre-commit", "installed": true, "version": "…", "outdated": false }, … ] }`).

### `bd config get/set/unset/list`

- **Python-обёртки:**
  - `def bd_config_get(key: str) -> dict`
  - `def bd_config_set(key: str, value: str) -> dict`
  - `def bd_config_unset(key: str) -> dict`
  - `def bd_config_list() -> dict`
- **JSON-ответ:** карты ключ→значение. Для `get/set/unset` возвращается `{ "key": "…", "value": "…" }` (при unset значение может быть пустой строкой), для list — полный конфиг.

### `bd rename-prefix`

- **Python-обёртка:** `def bd_rename_prefix(new_prefix: str, dry_run: bool = False) -> dict`
- **JSON-ответ:** `{ "old_prefix": "bd", "new_prefix": "kw", "renamed": 120, "dry_run": false }`.

### `bd doctor`

- **Python-обёртка:** `def bd_doctor(path: str | None = None, fix: bool = False, dry_run: bool = False, interactive: bool = False, output: str | None = None) -> dict`
- **JSON-ответ:** структура `doctorResult` `{ "path": "…", "overall_ok": bool, "checks": [ { "name": "…", "status": "ok|warn|error", "detail": "…", "fix": "…" } ], "cli_version": "…", "timestamp": "…", "platform": {…} }`.

### `bd detect-pollution`

- **Python-обёртка:** `def bd_detect_pollution() -> dict`
- **JSON-ответ:** счётчик и классификация тестовых задач `{ "polluted_count": n, "high_confidence": k, "medium_confidence": m, "issues": [ { "id": "…", "score": 0.95, "reason": "…" }, … ] }`.

### `bd repo add/remove/list/sync`

- **Python-обёртки:**
  - `def bd_repo_add(path: str, alias: str | None = None) -> dict`
  - `def bd_repo_remove(key: str) -> dict`
  - `def bd_repo_list() -> dict`
  - `def bd_repo_sync() -> dict`
- **JSON-ответ:**
  - add/remove: `{ "added": true, "key": "notes", "path": "../planning" }` или `{ "removed": true, … }`
  - list: `{ "primary": ".", "additional": { "notes": "../planning" } }`
  - sync: `{ "synced": true }` после ручной синхронизации всех настроенных репозиториев.

### `bd restore`

- **Python-обёртка:** `def bd_restore(ids: list[str]) -> list[dict]`
- **JSON-ответ:** восстановленные `Issue` (статус снова open/in_progress, поля удаления сброшены).

### `bd template` / `bd tips`

- **Python-обёртки:**
  - `def bd_template_list() -> list[str]`
  - `def bd_template_show(name: str) -> dict`
  - `def bd_tips() -> dict`
- **JSON-ответ:** список доступных шаблонов или содержимое конкретного шаблона `{ "name": "…", "body": "…" }`; `bd tips` возвращает словарь `{ "tips": ["…", "…"] }`. (Создание шаблонов через `bd template create` работает только в текстовом режиме.)

### `bd jira` / `bd message`

- **Python-обёртки:**
  - `def bd_jira(sync: bool = False, dry_run: bool = False) -> dict`
  - `def bd_message_send(target: str, body: str, *, subject: str = "", thread_id: str | None = None, importance: str = "normal", ack_required: bool = False) -> dict`
  - `def bd_message_inbox(limit: int = 20, unread_only: bool = False, urgent_only: bool = False) -> dict`
  - `def bd_message_read(message_id: str) -> dict`
  - `def bd_message_ack(message_id: str) -> dict`
- **JSON-ответ:** jira — отчёт по синхронизации `{ "synced": n, "dry_run": bool, "errors": ["…"] }`; message-send — подтверждение `{ "sent": true, "target": "…", "id": "msg-…" }`. Inbox/Read/Ack возвращают сообщения `Message` `{ "id": "msg-123", "subject": "…", "body": "…", "from_agent": "…", "created_at": "…", "importance": "high", "ack_required": true, "thread_id": "…", "read": bool, "acknowledged": bool }` либо списки таких объектов.

### `bd version`

- **Python-обёртка:** `def bd_version(daemon: bool = False) -> dict`
- **JSON-ответ:** `{ "version": "…", "build": "…", "commit": "…", "branch": "…" }`.

### `bd upgrade status/review/ack`

- **Python-обёртки:**
  - `def bd_upgrade_status() -> dict`
  - `def bd_upgrade_review() -> dict`
  - `def bd_upgrade_ack() -> dict`
- **JSON-ответ:**
  - status: `{ "upgraded": bool, "current_version": "…", "previous_version": "…", "changes_available": bool }`
  - review: `{ "from": "…", "to": "…", "releases": [ { "version": "…", "notes": "…" }, … ] }`
  - ack: `{ "acknowledged": true, "version": "…" }`.

## Команды без `--json`

Некоторые утилиты работают только в текстовом режиме и не имеют стабильного JSON-ответа:

- `bd init` — интерактивная инициализация базы/префикса; для автоматизации запускайте с `--quiet` и обрабатывайте обычный stdout.
- `bd prime` — печатает шпаргалку для агента (одноразовый вывод текста).
- `bd quickstart` и `bd workflow` — выводят пошаговые инструкции; обёртки могут просто возвращать текст stdout.
- `bd clean` — удаляет временные файлы из `.beads`; выдаёт текстовый отчёт.
- `bd setup claude|cursor|aider` — устанавливает интеграции с редакторами; выводит текстовые подсказки.
- `bd onboard` — генерирует шпаргалку по workflow в файл или stdout.
- `bd migrate-sync` — настраивает sync.branch и worktree; использует текстовый вывод для диагностики.

## Советы по сериализации в Python

- Все временные поля приходят в формате RFC3339. Используйте `datetime.fromisoformat` (Python 3.11+) либо `dateutil.parser`. 
- Булевы и числовые поля (`priority`, `compaction_level`, `estimated_minutes`) приходят уже типизированными в JSON. 
- Для обёрток удобно использовать `subprocess.run(["bd", "--json", …], capture_output=True, text=True, check=True)` и затем `json.loads` результата.

