# AI ServiceDesk Assistant: верхнеуровневый план

План основан на `sDa.txt` и учитывает CPU-only разработку, раннее выделение расширяемого интеграционного слоя, n8n как первый adapter endpoint и выполнение ранбуков только после согласования оператора в MVP.

## Правило языка

- Документация, пользовательские тексты, операторский интерфейс и консоль администратора ведутся только на русском языке.
- Технические идентификаторы не переводятся: API paths, JSON-поля, contract names, enum/status values, имена команд, имена файлов, tool names и adapter types.
- Если backend-строка отображается в UI оператору, она тоже должна быть написана по-русски.

## UX-правило административной консоли

- Для повторяемых сущностей используется паттерн: компактный список объектов -> один раскрытый редактор выбранного объекта -> расширенные настройки под свернутым блоком.
- Основной UI показывает человекочитаемые связи и действия, а технические идентификаторы остаются вторым слоем: payload, логи, tooltip или диагностика.
- JSON и JSON Schema остаются источником истины и контрольной точкой, но не являются основным ручным интерфейсом администратора.
- Подробные правила зафиксированы в `docs/UI_UX_APPROACH.md` и применяются ко всем новым переработкам UI.

## Базовые решения

- Оркестратор обращается только к LiteLLM, а не напрямую к vLLM, Ollama или будущему GPU backend.
- vLLM CPU используется в разработке как OpenAI-compatible smoke/integration backend, а не как финальный ориентир качества.
- n8n вводится рано, потому что запуск ранбуков дает высокую ценность, но n8n является только первым adapter и не должен стать единственным типом endpoint.
- Запуск ранбука является action tool и в MVP всегда требует явного согласования оператора.
- LangGraph решает, что должно произойти; Execution Policy, Tool Registry и Integration Dispatcher решают, можно ли вызывать интеграции и каким способом.
- Integration endpoints описываются каталогом/конфигурацией. AI может предложить tool action, но не может выбирать adapter, endpoint URL, execution mode или approval status.
- RAG добавляется после стабилизации основных контрактов решений и tool-call.
- Case API владеет жизненным циклом service desk case. Endpoint adapters, включая n8n, возвращают нормализованные результаты через синхронный tool result или callback API.

## Целевой жизненный цикл

```text
Operator UI
  -> FastAPI Case API
  -> AI Orchestrator
  -> LangGraph / Workflow Engine
  -> Execution Policy + Action Gate
  -> Tool Registry
  -> Integration Dispatcher
  -> Endpoint Adapter: mock / n8n_webhook / direct_http / queue / future
  -> External Systems
  -> Callback API / normalized tool result
  -> Case Store
  -> UI update / escalation / resolve / feedback
```

n8n является одним endpoint adapter в этом жизненном цикле. Он не является единственным путем исполнения и не должен протекать в AI decision contracts или case lifecycle contracts.

## Административная плоскость

Операторский интерфейс и консоль администратора являются разными интерфейсами с разными границами ответственности.

Операторский интерфейс:

- обрабатывает обращения пользователей;
- запускает анализ заявки;
- согласует или отклоняет действия по конкретному кейсу;
- сохраняет обратную связь по ответу AI;
- не меняет глобальную конфигурацию платформы.

Административная консоль:

- управляет поведением AI;
- управляет базой знаний;
- управляет ReAct-вызовами ИИ, endpoint adapters и интеграциями;
- управляет конфигурацией рабочего процесса;
- контролирует качество AI;
- управляет безопасностью, доступом и аудитом;
- не используется для обработки обращений пользователей.

Целевая административная архитектура:

```text
React-консоль администратора
  -> Admin API
  -> Config Registry / Audit / Quality
  -> Case API / Knowledge / Tool Registry / LiteLLM / Integration Dispatcher
```

Консоль администратора не должна напрямую менять business state кейса и не должна обходить Case API, Tool Registry, Execution Policy или Integration Dispatcher. Любое опасное административное действие должно иметь permission check, audit event и, где применимо, режим draft/validate/activate.

Начальный стек консоли администратора:

- React.
- TypeScript.
- Vite.
- Ant Design.
- React Query.
- React Router.

Начальные административные разделы:

- Панель обзора платформы.
- Сценарии обработки обращений как главный рабочий раздел администратора:
  - Сценарии;
  - 0. Слоты;
  - 1. Разрешение атрибутов;
  - 2. Классификация и маршрут;
  - 3. ReAct-планирование;
  - 4. ReAct-вызовы и матрица запуска;
  - 5. Решение и эскалация;
  - 6. Промпты;
- База знаний / RAG.
- Вызовы и интеграции:
  - Интеграции;
  - ReAct-вызовы ИИ;
  - Привязка операций.
- Конфигурация рабочего процесса.
- Модели.
- Пользователи, роли и права.
- Аудит и наблюдаемость.
- Контроль качества.
- Безопасность.

Немедленный приоритет административной консоли:

- Перестроить модель управления вокруг сценария обращения, а не вокруг разрозненных таблиц `tools`, `prompts`, `workflow` и `endpoints`.
- Закрепить пять шагов оркестратора как основную навигацию внутри сценария: слоты, классификация, ReAct-планирование, ReAct-вызовы, решение/эскалация.
- Добавить матрицу запуска ReAct-вызовов с привязкой к слотам, сценарию, ReAct-вызову ИИ, привязке операции, уровню риска и виду запуска.
- Перевести промпты из свободного каталога в prompt pack с обязательными блоками system prompt.
- Выполнить эти изменения до любых следующих расширений консоли администратора, observability, Vault, MFA, live n8n management и multi-model routing.

## Последовательность разработки

### Этап 0: базовый контур разработки

Цель: создать воспроизводимую локальную среду для CPU-only разработки.

Поставки:

- Docker Compose структура.
- PostgreSQL с возможностью pgvector.
- Redis.
- n8n в минимальном self-hosted режиме.
- LiteLLM и vLLM CPU за опциональным профилем.
- Шаблон окружения.
- Структура директорий проекта.
- Healthcheck и команды smoke-проверок.

Критерии выхода:

- `docker compose config` проходит.
- Порты и имена сервисов задокументированы.
- Основные сервисы можно запускать независимо от тяжелого LLM runtime.

### Этап 1: LLM gateway

Цель: открыть локальный OpenAI-compatible endpoint через LiteLLM.

Поставки:

- vLLM CPU с маленькой smoke-моделью `facebook/opt-125m`; `Qwen/Qwen3-0.6B` остается более поздним кандидатом по качеству.
- LiteLLM routing к vLLM.
- Smoke-проверка для `POST /v1/chat/completions`.
- Безопасные для CPU лимиты model length и KV cache.

Критерии выхода:

- FastAPI/LangGraph может обращаться к LiteLLM через стабильный model alias.
- У оркестратора нет прямой зависимости от vLLM-specific API.

### Этап 2: контракты AI-решений

Цель: сделать результаты AI машинно-валидируемыми до реализации сложных workflow.

Поставки:

- Расширяемые JSON Schema для:
  - envelope AI-решения.
  - типы решений: `answer_proposed`, `clarification_needed`, `escalation_needed`, `action_proposed`.
  - proposed actions.
  - результат execution policy.
- Слой валидации для model output.
- Policy boundary, отделяющий AI proposals от execution decisions.
- Failure path для invalid model output.

Критерии выхода:

- Invalid model output не может продвинуть workflow state.
- Model output может предлагать actions, но не может помечать их approved или auto-executable.
- Runbook proposals представлены как proposed actions; MVP policy мапит `start_systemcenter_runbook` на согласование оператора.
- Контракт позднее поддержит `auto_execute`, `dry_run`, `operator_approval`, `manual_only` и `blocked`.

### Этап 3: FastAPI и LangGraph skeleton

Цель: реализовать первый исполняемый service workflow.

Поставки:

- `POST /tickets/analyze`.
- Python FastAPI app.
- JSON Schema validation через Python `jsonschema`, где `contracts/` является source of truth.
- Модели ticket request/response без дублирования enum-логики в Python.
- Workflow state catalog и transition rules.
- Проверки обязательных полей.
- Начальный classification node.
- Decision node.
- Execution policy node.
- Mock tool node.
- Расширяемый `workflow_state` для ранбуков, invalid model output, clarification, escalation и будущих auto-execution states.

Критерии выхода:

- API валидирует AI decisions, proposed actions, execution policy results и workflow states по JSON Schema из `contracts/`.
- Backend обрабатывает answer, clarification, escalation и action proposals.
- Approval или auto-execution mode выводится из policy, а не из raw model output.
- Workflow state выводится из catalog и transition rules, а не из hardcoded enum.
- Новые workflow state IDs можно добавлять данными config/schema без изменения core workflow code.
- Сценарии можно тестировать без реальных внешних систем.

### Этап 4: основа интеграционного слоя

Цель: рано ввести расширяемый integration layer. n8n является первым реальным adapter, но не единственной моделью endpoint.

Поставки:

- Минимальные контракты Tool Registry:
  - tool definition.
  - tool invocation.
  - normalized tool result.
- Контракты integration endpoint:
  - endpoint definition.
  - adapter type.
  - auth reference.
  - операция mapping.
- Endpoint catalog минимум с:
  - `mock` endpoint для локальных тестов.
  - `n8n_webhook` endpoint для запуска ранбуков.
- Tool catalog минимум с:
  - `start_systemcenter_runbook`, привязанным к configurable endpoint.
  - одним read-only diagnostics stub, привязанным к `mock`.
- Python Integration Dispatcher.
- Adapter interface.
- `mock` adapter.
- `n8n_webhook` adapter.
- Минимальный n8n webhook workflow для запуска ранбука.
- Token-based защита webhook.
- Общий normalized JSON result contract для всех adapters.

Критерии выхода:

- Оркестратор никогда не вызывает n8n напрямую.
- `start_systemcenter_runbook` проходит через Tool Registry и Integration Dispatcher к configured endpoint.
- Endpoint можно заменить с n8n на mock или будущий direct API через config/catalog data.
- Action-вызовы требуют execution policy и согласование оператора до вызова dispatcher.
- n8n workflows можно mock-ать без изменения orchestrator workflow code.
- AI output не может выбирать endpoint URL, adapter type или execution mode.

### Этап 5: расширение Tool Registry и policy

Цель: расширить tool catalog и policy model после стабилизации dispatcher boundary.

Поставки:

- Полные declarative tool definitions для MVP tools.
- Tool types: `read_only` и `action`.
- Tool-specific input/output schemas.
- Timeout и retry policies.
- Approval policy per tool.
- Политика автоисполнения для каждого ReAct-вызова.
- Adapter-specific операция mapping.
- Опциональные будущие adapters: `direct_http` или queue-based invocation.
- Tool trace и normalized error fields.

Начальные tools:

- `check_zabbix_status` как `read_only`.
- `query_cmdb_object` как `read_only`.
- `get_service_owner` как `read_only`.
- `search_known_incidents` как `read_only`.
- `start_systemcenter_runbook` как `action` с обязательным согласованием.

Критерии выхода:

- LLM output не может напрямую вызывать external systems.
- Action-вызовы требуют policy evaluation до Integration Dispatcher.
- MVP policy требует согласование оператора для `start_systemcenter_runbook`; позднее policy может разрешить low-risk auto-execution по allowlist.
- Добавление или замена endpoint adapter не требует изменения AI decision contracts.

### Этап 6: ранбуки с согласованием оператора

Цель: дать раннюю высокую ценность через backend-owned action gate и сохранить human approval gate для MVP runbooks.

Поток:

1. Оператор отправляет заявку.
2. LangGraph анализирует заявку и read-only diagnostics.
3. AI возвращает `action_proposed` с runbook proposal.
4. Backend оценивает execution policy.
5. Backend создает persisted action gate record.
6. Backend возвращает `operator_approval` и `approval_requests` для MVP runbooks.
7. UI показывает параметры предложенного действия, expected effect и risk notes; технический код ранбука остается частью параметров вызова или endpoint payload, а не отдельным полем заявки.
8. Оператор согласует или отклоняет по `approval_id`.
9. Backend вызывает Integration Dispatcher только после approval, используя сохраненные action и policy result.
10. Dispatcher вызывает configured endpoint; в MVP это mock или n8n webhook.
11. Endpoint возвращает normalized result через adapter.
12. Заявка получает runbook result, rejection state или escalation summary.

Архитектурные решения:

- Использовать action-gate record вместо UI-supplied approval flag.
- Поддержать `gate_type=operator_approval` сейчас и зарезервировать `gate_type=auto_policy` для будущего auto-execution.
- Хранить action-gate contracts в JSON Schema и валидировать их в Python backend.
- Хранить MVP gate records в local SQLite через replaceable store abstraction.
- Workflow states и transitions остаются catalog-driven.

Критерии выхода:

- Модель может предложить runbook, но не может его выполнить.
- У каждого runbook execution есть approval record.
- Каждый внешний runbook call проходит через Tool Registry и Integration Dispatcher.
- Public dispatch API отклоняет client-supplied `approved_by_operator=true`.
- Отклоненные approvals не вызывают integrations.
- Повторное approval decision не может выполнить тот же runbook дважды.

### Этап 7: загрузка базы знаний и минимальный RAG

Цель: использовать внутренние знания из нескольких источников, не блокируя раннюю ценность tools/runbooks.

Поставки:

- Каталог источников знаний для local files, corporate wiki, ServiceDesk KB, CMDB exports, Git repositories и будущих API-источников.
- Интерфейс connector для загрузки источников.
- Реализованный connector `local_files` для `.md` и `.txt` документов разработки.
- Отключенные/config-ready skeletons для remote sources, например corporate wiki.
- Нормализованный контракт документа, общий для всех источников.
- Chunking job и локальный лексический индекс для CPU-only разработки.
- pgvector-backed retrieval как целевой backend за тем же интерфейсом retriever, не жесткая зависимость локальной разработки.
- Действие перестроения базы знаний, запускаемое оператором, с manifest, status, counts, errors и timestamps.
- Retriever node в LangGraph/FastAPI workflow, использующий последний успешный индекс.
- Ссылки на источники в ответе оператору.

Архитектурные решения:

- `knowledge/` является только локальным development source, а не всей RAG architecture.
- `/tickets/analyze` не должен синхронно загружать произвольные wiki/API pages.
- Перестроение базы знаний является явным действием оператора, а не автоматикой на каждый анализ заявки.
- Analysis использует latest successful index и деградирует корректно при retrieval failure.
- Corporate wiki и другие remote sources загружаются через connectors и нормализуются до индексирования.

Критерии выхода:

- В config/catalog data можно представить несколько source types.
- Local source ingestion и retrieval работают end to end.
- Disabled remote source skeletons не ломают rebuild или analysis.
- Rebuild создает validated index manifest.
- Workflow может цитировать найденные документы.
- RAG failures приводят к escalation/clarification или обычному ответу, а не к поломанному response.

### Этап 8: минимальный Operator UI

Цель: предоставить рабочий end-to-end manual workflow.

Поставки:

- Static operator UI, отдаваемый FastAPI на `/operator`.
- Форма ввода заявки.
- Отображение AI decision.
- Отображение clarification.
- Отображение escalation.
- Контролы approve/reject для runbook.
- Отображение integration/runbook execution result.
- Контрол перестроения базы знаний оператором.
- Отображение knowledge index status.
- Copy-to-service-desk text.
- Tool call и RAG trace view.

Архитектурные решения:

- UI этапа 8 остается static и backend-served, без отдельного frontend toolchain.
- Использовать тот же approval API, что и backend smoke tests; не разрешать UI-side execution bypass.
- Feedback collection вынесен из scope этапа 8.
- RBAC/login вынесен в этап 11.

Критерии выхода:

- Оператор может вручную обработать заявку от input до copyable final result.
- Runbook approval видим и явен.
- Перестроение базы знаний видно и явно выполняется как действие оператора.

### Этап 9: обратная связь и цикл оценки

Цель: собрать данные обратной связи для будущей оценки качества перед расширением agents.

Поставки:

- Контракты JSON Schema для обратной связи.
- Operator feedback: `correct`, `incorrect`, `edited`.
- Сохраненный ticket input.
- Сохраненный decision output.
- Сохраненные RAG hits.
- Сохраненные tool outputs.
- Опциональный snapshot результата согласования или ранбука.
- SQLite-backed feedback store.
- Контролы обратной связи в Operator UI.
- Exportable regression dataset seed.

Архитектурные решения:

- Обратная связь должна ссылаться на точный analysis snapshot, который видел оператор.
- `edited` feedback требует corrected response.
- Export format: JSONL evaluation cases для будущих regression runs.
- Full evaluation runner, feedback history UI и quality analytics отложены в этап 10.5.
- RBAC, full audit log и automated model training вне scope.

Критерии выхода:

- Обратную связь можно сохранить и экспортировать без изменения ticket workflow state.
- Exported feedback содержит достаточно контекста, чтобы стать curated evaluation set в этапе 10.5.

### Этап 10: жизненный цикл кейса и callback API

Цель: ввести durable case lifecycle boundary перед усилением безопасности пилотного контура.

Поставки:

- Контракты кейсов:
  - `case-record`.
  - `case-event`.
  - `case-timeline`.
  - `integration-callback`.
- SQLite-backed `CaseStore` abstraction.
- Case API:
  - `POST /cases` или compatible case creation через `/tickets/analyze`.
  - `GET /cases/{case_id}`.
  - `GET /cases/{case_id}/timeline`.
  - callback endpoint для normalized integration results.
- Persisted case lifecycle state:
  - ticket input.
  - current workflow state.
  - AI decision snapshot.
  - RAG trace.
  - tool trace.
  - action gate references.
  - feedback references.
  - final resolution, escalation или failed state.
- Correlation identifiers для async work:
  - `case_id`.
  - `ticket_id`.
  - `action_id`.
  - `invocation_id`.
  - `endpoint_id`.
- Callback handling для long-running endpoint adapters.
- UI polling case status и timeline.
- Smoke scenario для async-style runbook result:
  - create/analyze case.
  - create pending action gate.
  - simulate endpoint callback.
  - update case state and timeline.
  - read updated case through API/UI.

Архитектурные решения:

- Case API владеет lifecycle state; Integration Dispatcher и adapters возвращают только normalized tool results/callbacks.
- n8n callback payloads нормализуются на границе и не становятся core case schema.
- Synchronous adapter results и asynchronous callbacks используют один normalized result model.
- UI начинает с polling; WebSocket/SSE не нужны на этом этапе.
- Hardening идет после стабилизации lifecycle/callback boundary.

Критерии выхода:

- Case можно создать, прочитать и обновить через normalized workflow events.
- Long-running endpoint results могут обновить case после initial acceptance.
- Case timeline показывает analysis, approval, dispatch, callback/result и feedback events.
- Existing ticket-oriented endpoints остаются compatible или мапятся на case records.
- Добавление или замена endpoint adapter не требует изменения case lifecycle contracts.

### Этап 10.5: основа Admin API и операции оценки

Цель: создать backend-основу административной плоскости до полноценной React-консоли администратора и завершить отложенный на этапе 9 evaluation loop поверх durable case lifecycle.

Поставки:

- Пространство Admin API, например `/admin/*`, без обхода существующих доменных API.
- Endpoint'ы панели обзора только для чтения:
  - счетчики по кейсам;
  - ожидающие согласования;
  - ошибки ReAct-вызовов и интеграций;
  - статус индекса RAG;
  - статус LiteLLM/model backend;
  - счетчики обратной связи.
- Административные endpoint'ы базы знаний:
  - просмотр каталога источников;
  - действие перестроения;
  - манифест индекса;
  - просмотр фрагментов;
  - тестовый поиск.
- Endpoint'ы инвентаризации каталогов:
  - ReAct-вызовы ИИ;
  - точки интеграции;
  - состояния workflow и правила переходов;
  - конфигурация маршрутизации моделей.
- Реализация evaluation runner с существующими контрактами обратной связи:
  - `evaluation-run`.
  - `evaluation-result`.
- Процесс переноса raw feedback records в подготовленный набор оценочных кейсов.
- Захват метаданных версии:
  - model.
  - prompt/config.
  - knowledge index.
  - tool registry/config.
- Идемпотентность обратной связи и поиск дублей.
- Ссылки timeline кейса на события обратной связи и оценки.

Архитектурные решения:

- Admin API сначала работает в режимах только для чтения или safe-action-first; изменение конфигурации идет позже через draft/validate/activate.
- CaseStore и case timeline являются durable source для snapshots; raw feedback этапа 9 остается import/export input.
- Запуски оценки по умолчанию используют dry-run, mock или явно sandboxed execution policy и не запускают реальные внешние действия.
- Подготовленные оценочные кейсы отделены от raw feedback, чтобы шумные заметки оператора не становились регрессионными проверками автоматически.
- Административные действия уже пишут actor/version metadata, даже если полноценный RBAC и усиление аудита завершаются на этапе 11.

Критерии выхода:

- Admin API дает безопасный обзор платформы без прямого изменения business state кейса.
- Перестроение базы знаний и тестовый поиск доступны как административные действия.
- Подготовленный набор оценочных кейсов можно повторно запускать против текущего workflow.
- Результаты оценки показывают pass/fail status и differences для изменений модели, промпта, RAG и ReAct-вызовов.
- События обратной связи и оценки видимы в timeline кейса.

### Этап 11: RBAC, аудит и основа безопасности

Цель: закрыть административные и операторские действия permission checks, audit log и базовой безопасностью до полноценной консоли администратора.

Поставки:

- Пользователи, роли и права.
- Начальные роли:
  - `admin`;
  - `operator`;
  - `support_l1`;
  - `support_l2`;
  - `readonly`.
- Модель прав:
  - `cases.read`;
  - `cases.operate`;
  - `approvals.decide`;
  - `callbacks.write`;
  - `feedback.write`;
  - `knowledge.read`;
  - `knowledge.manage`;
  - `tools.read`;
  - `tools.manage`;
  - `prompts.read`;
  - `prompts.manage`;
  - `workflow.read`;
  - `workflow.manage`;
  - `models.read`;
  - `models.manage`;
  - `evaluation.run`;
  - `audit.read`;
  - `security.manage`.
- Audit log для:
  - действий оператора;
  - действий администратора;
  - вызова ReAct-вызовов;
  - решений по согласованиям;
  - обработки callback;
  - изменений конфигурации.
- Ссылки на секреты:
  - API-ключи;
  - webhook-секреты;
  - ссылки на учетные данные интеграций;
  - будущая интеграция с Vault.
- Управление сессиями.
- Лимиты запросов.
- Совместимость с существующим `/healthz`.

Архитектурные решения:

- RBAC применяется на Admin API и Operator API.
- Audit log пишет факт действия и actor, но не сохраняет секретные значения.
- Управление секретами хранит ссылки, а не секреты в plain config.
- Action-вызовы не могут обходить approvals даже при наличии admin role.
- Callback endpoint'ы требуют token и пишут audit events.
- На этапе 11 используется dev-header auth; production identity provider, MFA, Vault и IP restrictions остаются следующими шагами.

Критерии выхода:

- Все опасные operator/admin действия требуют permission.
- Активность оператора, администратора и интеграций трассируема.
- Callback endpoints защищены и auditable.
- Case lifecycle changes attributable.

### Этап 12: MVP React-консоли администратора

Цель: дать администратору отдельную консоль управления платформой поверх Admin API.

Поставки:

- Консоль администратора на `/admin`.
- Статический MVP, обслуживаемый backend, без npm-зависимостей для текущего локального контура.
- Левое меню, верхняя панель и строка статуса операций закреплены при прокрутке длинных административных форм.
- Целевая миграция интерфейса на React/TypeScript/Vite/Ant Design поверх тех же Admin API.
- Frontend-стек:
  - React.
  - TypeScript.
  - Vite.
  - Ant Design.
  - React Query.
  - React Router.
- Раздел "Панель обзора":
  - количество обращений;
  - auto-resolve;
  - escalation;
  - ожидающие согласования;
  - ошибки интеграций;
  - состояние LLM;
  - состояние индексации RAG;
  - использование ReAct-вызовов.
- Раздел "База знаний / RAG":
  - источники;
  - перестроение индекса;
  - манифест;
  - фрагменты;
  - тестовый поиск;
  - заметки о качестве поиска.
- Раздел "Вызовы и интеграции":
  - подраздел "Интеграции";
  - подраздел "ReAct-вызовы ИИ";
  - подраздел "Привязка операций";
  - группировка подключений по adapter type;
  - создание, модификация и удаление подключений;
  - создание, модификация и удаление операций внутри подключения;
  - создание, модификация и удаление ReAct-вызовов ИИ;
  - текущая привязка `ReAct-вызов ИИ -> подключение -> операция` в отдельном меню;
  - таймауты, повторы и уровень риска;
  - блокировка удаления объектов, которые используются в сценариях, матрицах, профилях разрешения, каналах или n8n workflow;
  - история вызовов и callbacks.
- Раздел "Модели":
  - реальная настройка подключений LiteLLM;
  - профиль `vLLM CPU` для локальной CPU-разработки через OpenAI-compatible endpoint;
  - профиль `OpenAI API` с реальной моделью и token только через переменную окружения/секрет;
  - создание, модификация и удаление пользовательских подключений LiteLLM;
  - поддержка provider-prefixed model в alias для подключений без отдельной записи в `litellm.yaml`;
  - стабильное хранение настроек каждого подключения отдельно от активного выбора;
  - быстрое переключение между подключениями без повторного ввода параметров;
  - выбор активного подключения;
  - alias модели по умолчанию;
  - ввод значения секрета без последующего отображения: `параметр скрыт` при наличии и `параметр не заполнен` при отсутствии;
  - маршрутизация задач: ответы, классификация, суммаризация, выбор ReAct-вызовов, разрешение атрибутов;
  - fallback между подключениями;
  - context length, temperature, max tokens, timeout и rate limits.
- Раздел "Контроль качества":
  - список обратной связи;
  - подготовленный набор оценочных кейсов;
  - статус запусков оценки;
  - сводка регрессии.
- Раздел "Аудит":
  - действия администратора;
  - действия оператора;
  - вызовы ReAct-вызовов;
  - события workflow;
  - ошибки.
- Раздел "Безопасность":
  - пользователи;
  - роли;
  - права;
  - ссылки на API-ключи;
  - ссылки на webhook-секреты.

Архитектурные решения:

- Консоль администратора использует только Admin API и не обращается напрямую к хранилищу.
- Любой тестовый запуск ReAct-вызова использует `mock`, `dry_run` или явно sandboxed endpoint.
- Редактирование production-конфигурации не выполняется прямой записью; используется draft/validate/activate.
- Управление workflow n8n является частью интеграций, а не отдельным владельцем business state.

Критерии выхода:

- Администратор может наблюдать состояние платформы и запускать безопасные административные действия.
- Перестроение базы знаний и тестовый поиск доступны из консоли администратора.
- ReAct-вызовы, endpoints, промпты, модели, качество и аудит доступны минимум в режиме только для чтения или безопасного действия.
- Операторский интерфейс и консоль администратора разделены по маршрутам, ролям и permissions.

### Этап 12.5: сценарная модель оркестратора и связная консоль администратора

Цель: устранить разрозненность консоли администратора и сделать основной единицей управления сценарий обработки обращения, проходящий через пять шагов оркестратора.

Этот этап выполняется раньше всех остальных расширений административной консоли: observability, Vault, MFA, live n8n management, multi-model routing и прочие улучшения не начинаются, пока сценарная модель не закреплена контрактами и UI.

Поставки:

- Новые домены конфигурации в Config Registry:
  - `service_scenarios`;
  - `slot_schemas`;
  - `classification_routes`;
  - `orchestrator_policy`;
  - `tool_launch_matrix`;
  - `prompt_packs`;
  - `escalation_policies`.
- JSON Schema для каждого нового домена; схемы остаются источником истины для Python backend через `jsonschema`.
- Главный раздел консоли администратора "Сценарии обработки -> Сценарии":
  - выбор сценария обращений через выпадающее меню;
  - операции "Создать", "Модифицировать", "Удалить";
  - поля редактирования для создания и модификации сценария;
  - выбор подготовленных блоков `0-5` через выпадающие поля: профиль разрешения атрибутов выбирается в схеме слотов, сам сценарий выбирает схему слотов, маршрут, ReAct-политику, матрицу ReAct-вызовов, пакет промптов и политику эскалации;
  - статус готовности сценария;
  - связанные слоты, маршруты, ReAct-вызовы, prompt pack и escalation policy;
  - предупреждения о неполных или несогласованных связях.
- Отдельные подпункты меню с сохраненными номерами шагов оркестратора:
  - "0. Слоты";
  - "1. Разрешение атрибутов";
  - "2. Классификация и маршрут";
  - "3. ReAct-планирование";
  - "4. ReAct-вызовы и матрица запуска";
  - "5. Решение и эскалация";
  - "6. Промпты".
- Подпункты `0-5` являются самостоятельными каталогами подготовленных блоков:
  - в самих блоках нет управляемой связи со сценариями;
  - у каждого блока есть поле "Название";
  - доступны операции "Создать", "Модифицировать", "Удалить";
  - блок "Где используется" показывает сценарии, которые выбрали этот блок;
  - удаление запрещено, пока блок используется сценарием или схемой слотов.
- Шаг 1, слоты:
  - добавление, изменение и удаление отдельных слотов через форму;
  - существующие слоты в интерфейсе свернуты по умолчанию, кнопка удаления находится справа в строке слота, новый слот раскрывается после добавления;
  - технические идентификаторы сценария, схемы слотов, маршрута, политик, запусков ReAct-вызовов и prompt pack не выводятся как поля управления, но сохраняются в payload для связей, логов и audit trail;
  - пояснения под основными полями слота, чтобы администратор понимал смысл `slot_id`, `display_name`, `priority_group`, `required`, `fill_method` и контекстных полей `user_question`, `case_source_ref`, `extraction_instruction`, `resolution_profile_id`, `fallback_question`, `operator_hint`;
  - переопределение порогов уверенности для отдельного слота доступно только в свернутом блоке исключений и включается чекбоксом `Включить`; по умолчанию слот наследует системные и сценарные значения, а UI показывает системные значения в неактивной базе;
  - без ручного JSON-редактора для состава слотов;
  - required slots;
  - auto-fill slots;
  - порядок вопросов;
  - приоритет "кто -> что -> когда";
  - timeout 3 минуты для напоминания;
  - timeout 8 минут для черновика заявки и сохранения контекста;
  - выбор готового профиля разрешения атрибута из каталога `1. Разрешение атрибутов`.
- Операторский тестовый прогон:
  - режимы `Проверка конфигурации`, `С моделью`, `С моделью и безопасными интеграциями`, `Отладочный запуск с подтверждениями`;
  - явные флаги разрешений для LLM, read-only интеграций, mock-интеграций и action-вызовов с подтверждением;
  - в режиме `С моделью` выполняется `llm_extraction` и показываются значение, confidence, решение порога и причина;
  - базовый режим остается безопасным и не вызывает LLM или внешние системы;
  - трасса прогона показывает выполненные, пропущенные и заблокированные шаги.
- Шаг 2, классификация и маршрут:
  - правила классификации с позитивными и негативными признаками;
  - LLM few-shot fallback;
  - пороги confidence;
  - маршрут на автоагента, агента с подтверждением, человека с подсказкой, approver или Major Incident;
  - матрица "категория -> приоритет -> маршрут -> действие".
- Шаг 3, ReAct-планирование:
  - системные пороги уверенности для автопринятия, уточнения, передачи оператору и минимального качества LLM extraction;
  - сценарные override-пороги скрыты в свернутом блоке, включаются чекбоксом `Включить` и используются только для исключений;
  - лимит итераций;
  - правило двух ошибок ReAct-вызовов подряд;
  - стоп-условия;
  - разрешенные группы действий ReAct для сценария;
  - связь с workflow states и transition rules.
- Шаг 4, ReAct-вызовы:
  - матрица запуска ReAct-вызовов с привязкой к сценарию и слотам;
  - `required_slots` для каждого запуска;
  - единый UI-параметр "Режим исполнения сейчас";
  - служебные `parameter_bindings` из slot/constant/secret/case/context без ручного JSON-редактора в UI;
  - внутренние `execution_level` и `target_execution_level` синхронизируются из выбранного вида запуска;
  - роль согласования;
  - подключение;
  - risk level;
  - audit/log флаги;
  - stop-on-error политика.
- Шаг 5, решение и эскалация:
  - условия автозакрытия;
  - режим ожидания ответа пользователя;
  - правила остановки SLA;
  - автозакрытие через 24 часа;
  - передача в канал эскалации;
  - Major Incident threshold;
  - состав escalation package.
- Раздел консоли администратора "Сценарии обработки -> 6. Промпты":
  - выбор пакета промптов через выпадающее меню;
  - операции "Создать", "Модифицировать", "Удалить";
  - редактирование блока "Пакет промптов: обязательные блоки";
  - защита от удаления пакета, если активный сценарий продолжает на него ссылаться.
- Prompt pack вместо свободного списка промптов:
  - `role_context`;
  - `behavior_principles`;
  - `slot_schemas`;
  - `classification_confidence`;
  - `react_planning`;
  - `tool_rules`;
  - `escalation_response`.
- Preview итогового system prompt доступен на уровне API и отладочных данных, но не дублируется как отдельный блок в основном UI.
- Отдельный preview системного промпта не нужен в основном UI, если обязательные блоки prompt pack уже показаны как редактируемый шаблон в меню "Сценарии обработки -> 6. Промпты".
- Переработанная панель оператора:
  - одно неструктурированное поле текста заявки вместо прежней формы с фиксированными полями;
  - выбор сценария обработки;
  - галочка "Тестовый прогон сценария" для dry-run проверки слотов, классификации и доступных инструментов;
  - раскрытая карта пяти шагов оркестратора для выбранного сценария;
  - вопросы обогащения заявки по одному недостающему слоту;
  - отображение текущей готовности ReAct-вызовов по матрице запуска и привязке к слотам;
  - сохранение действующих операторских функций: анализ, согласование action tools, feedback, RAG/tool trace и timeline кейса.

Архитектурные решения:

- `service_scenarios` становится верхним уровнем управления, а текущие `tools`, `integration_endpoints`, `workflow_states`, `workflow_transitions`, `prompts`, `model_routing` и `n8n_workflows` становятся связанными нижнеуровневыми справочниками.
- `service_scenarios` собирает сценарий из независимых блоков: сами `slot_schemas`, `classification_routes`, `orchestrator_policy`, `tool_launch_matrix` и `escalation_policies` не содержат управляемой ссылки на сценарий.
- Матрица запуска ReAct-вызовов строже каталога ReAct-вызовов: ReAct-вызов нельзя предложить к запуску в сценарии, если не заполнены required slots или нет корректного parameter binding.
- В UI матрицы ReAct-вызовов используется единый "Режим исполнения сейчас"; backend сохраняет его в `execution_level` и `target_execution_level` для совместимости с контрактом матрицы.
- В UI матрицы ReAct-вызовов карточка запуска группируется по смыслу: "ReAct-вызов ИИ" выбирает бизнес-действие, "Текущая привязка операции" показывает выбранное подключение/операцию без редактирования, "Параметры вызова" содержит маппинг параметров из схемы вызова на источники `slot/case/context/constant/secret`, а "Контроль запуска" содержит режим исполнения сейчас, риск, согласование, аудит, логирование и stop on error. Endpoint, adapter, operation request schema, timeout и технический вызов редактируются в подразделах "Вызовы и интеграции -> Интеграции" и "Вызовы и интеграции -> Привязка операций".
- Подраздел "Вызовы и интеграции -> Интеграции" является техническим справочником подключений: подключения сгруппированы по adapter type, операции редактируются внутри подключения.
- Подраздел "Вызовы и интеграции -> ReAct-вызовы ИИ" хранит бизнес-политику вызова, JSON Schema параметров и JSON Schema результата.
- Подраздел "Вызовы и интеграции -> Привязка операций" связывает ReAct-вызов ИИ с одной текущей операцией endpoint по модели `tool -> endpoint_id -> operation_id`; новая привязка заменяет старую.
- `required_slots` в матрице ReAct-вызовов не редактируется вручную в UI и вычисляется из всех `parameter_bindings` вида `slot:<slot_id>`. Slot schema не знает о ReAct-вызовах; переиспользование матрицы возможно только для сценариев с совместимыми слотами или через будущий слой scenario override.
- Prompt pack не редактируется как один свободный system prompt; UI обязан разделять его на семь обязательных блоков в меню "Сценарии обработки -> 6. Промпты".
- Блоки prompt pack, зависящие от слотов, классификации, ReAct и tools, должны собираться из структурированной конфигурации, чтобы промпты не расходились с реальным поведением backend.
- Все изменения проходят через существующий процесс draft/validate/regression/activate.
- В разделе "Сценарии обработки -> Сценарии" не используется отдельная таблица "Карта сценариев", если доступен выпадающий выбор сценария.
- Каждый из пяти шагов сценария редактируется через свой домен Config Registry, а не через ручную правку JSON в разделе общей конфигурации.
- Операторская панель читает сценарную модель через отдельные operator endpoint'ы с правом `cases.operate`; ей не требуются административные permissions.

Валидации перед активацией:

- Каждый сценарий имеет slot schema.
- Каждый required slot либо задается вопросом, либо имеет auto-fill источник.
- Порядок вопросов не нарушает приоритет "кто -> что -> когда".
- Каждый route ссылается на допустимое состояние workflow и выбирается сценарием через `classification_route_id`.
- Каждая матрица ReAct-вызовов выбирается сценарием через `tool_launch_matrix_id`.
- Каждый запуск ReAct-вызова ссылается на существующий tool; подключение и операцию автоматически берутся из текущей привязки `tool.endpoint_bindings`.
- Каждый параметр запуска ReAct-вызова имеет binding на slot, constant, secret, case или context.
- `execution_level=auto` запрещен для blocked/high-risk действий без явного разрешения policy.
- Каждый prompt pack содержит все семь обязательных блоков.
- Escalation package содержит слоты, историю ReAct, результаты ReAct-вызовов, гипотезу причины, остаток SLA и текст уведомления пользователя.
- Major Incident threshold задан явно для сценариев, где возможен массовый инцидент.

Критерии выхода:

- Администратор видит сценарий как связную карту: слоты, классификация, ReAct, ReAct-вызовы, решение и эскалация.
- Оператор видит тот же сценарий как рабочую карту обработки заявки и может обогащать обращение вопросами по слотам до запуска анализа.
- Нельзя активировать сценарий с неполной матрицей слотов, tool bindings или prompt pack.
- В операторском UI можно включить тестовый прогон сценария без реального вызова внешних систем.
- Текущий MVP остается с подтверждением оператора для action tools, но в конфигурации уже видны целевые места будущего автоисполнения.
- Разрозненные разделы tools/prompts/workflow остаются доступными, но перестают быть основным способом настройки поведения AI.

### Этап 12.6: профили разрешения атрибутов для заполнения слотов

Цель: заменить простое поле "Источник" в слоте на расширяемую модель "атрибут -> правило заполнения", где схема слотов выбирает готовое правило, а сами правила управляются в отдельном разделе консоли администратора.

Поставки:

- Новый домен Config Registry `attribute_resolution_profiles`.
- JSON Schema для профилей разрешения атрибутов; схема остается источником истины для Python backend через `jsonschema`.
- Новый пункт меню консоли администратора "Сценарии обработки -> 1. Разрешение атрибутов".
- Операции в меню "Сценарии обработки -> 1. Разрешение атрибутов":
  - "Создать";
  - "Модифицировать";
  - "Удалить".
- UI-обвязка CRUD:
  - выбор профиля из выпадающего списка;
  - форма создания профиля;
  - форма модификации профиля;
  - редактирование шагов профиля через раскрывающиеся карточки с полями, без JSON textarea;
  - форма удаления с подтверждением;
  - общий процесс draft/validate/regression/activate;
  - audit event для каждой операции.
- Структура профиля разрешения атрибута:
  - название;
  - статус;
  - сценарий для выбора слотов в UI;
  - целевой атрибут/слот, выбираемый из доступных слотов сценария;
  - выходные слоты, выбираемые из доступных слотов сценария;
  - признаки для поиска;
  - операция разрешения;
  - правила оценки результата операции;
  - матрица решений;
  - правила уточняющих вопросов и передачи человеку;
  - уточняемые атрибуты и пакет передачи человеку, выбираемые в UI из доступных слотов сценария;
  - лимит попыток уточнения;
  - внутренние confidence thresholds профиля, скрытые в UI под свернутым блоком и применяемые только для исключений конкретного resolver;
  - fallback: спросить пользователя, передать оператору, эскалировать или оставить незаполненным;
  - audit/log флаги.
- Пример профиля "Поиск логина в AD по ФИО":
  - извлечь признаки личности из текста обращения;
  - получить результаты поиска из AD;
  - если найден один пользователь и confidence достаточный, заполнить `user_login`;
  - если найдено несколько совпадений или ничего не найдено, запросить должность, подразделение или табельный номер;
  - если результат неоднозначен после лимита попыток, передать человеку.
- Профиль истории заявок как отдельная настройка:
  - статусы заявок;
  - период;
  - минимальная оценка качества;
  - similarity threshold;
  - разрешенные поля;
  - исключенные категории.
- Интеграция с ReAct-вызовами:
  - tool остается runtime-сущностью исполнения на уровне контрактов;
  - профиль разрешения атрибута ссылается на ReAct-вызов как операцию разрешения;
  - сценарий не группирует ReAct-вызовы вручную и не показывает полный каталог tools для заполнения слота.
- Изменение редактора слотов в разделе "Сценарии":
  - убрать прямой выбор "Источник" как основную настройку;
  - добавить "Как получить значение слота": вопрос пользователю, LLM extraction, профиль разрешения атрибута, ручное заполнение оператором;
  - показывать только поля, допустимые для выбранного способа заполнения;
  - для `user_question` редактировать `user_question`, для `llm_extraction` - `extraction_instruction` и `examples`, для `resolution_profile` - `resolution_profile_id` и `fallback_question`, для `operator_manual` - `operator_hint`;
  - если выбран профиль, показывать только выпадающий список профилей, которые могут выдавать целевой слот;
  - показывать read-only сводку профиля: входы, выходы, fallback, audit и требуемые уточнения.
- Изменение операторской панели:
  - при недостающем слоте показывать решение resolver-профиля;
  - если профиль требует уточнение, оператор видит конкретный вопрос и варианты уточняемых полей;
  - оператор не выбирает tool вручную для заполнения атрибута.

Архитектурные решения:

- Slot filling не является свободным ReAct loop; для заполнения атрибутов используется детерминированный resolver с матрицей решений.
- Tools не удаляются из архитектуры: они остаются исполняемыми адаптерами, но не являются основной сущностью настройки слота.
- `llm_extraction` считается встроенным способом заполнения; AD, CMDB, IAM и внешние системы подключаются через профили разрешения атрибутов.
- `case` ("из данных обращения") и история заявок остаются future-источниками и возвращаются в активный UI только после реализации исполнения и правил доступа.
- Профили разрешения атрибутов не владеют business state кейса; результат заполнения проходит через Case API и workflow state.
- Эффективные пороги уверенности наследуются в порядке: системные значения -> override сценария -> override слота. Профиль разрешения атрибута может переопределить их только для исключений своего resolver.
- Сценарий остается компактным: он выбирает схему слотов, а схема слотов выбирает готовые правила заполнения без описания endpoint, retry, tool execution и фильтров истории внутри сценария.

Валидации перед активацией:

- Каждый профиль имеет уникальный технический ключ, но ключ не выводится как поле управления в UI.
- Слот может ссылаться только на существующий профиль, который выдает этот слот как output.
- В консоли администратора `target_slot_id` и `output_slots` набираются из схемы слотов выбранного сценария; технически существующие значения вне выбранного сценария показываются как отсутствующие, чтобы администратор явно исправил настройку.
- В консоли администратора `clarification_policy.ask_for_attributes` и `handoff_policy.package` также выбираются из схемы слотов выбранного сценария; выбранные слоты автоматически объявляются как атрибуты профиля, если они еще не были объявлены.
- Чтобы не возникал цикл создания "слот -> профиль -> слот", UI поддерживает bootstrap:
  - в `1. Разрешение атрибутов` профиль можно создать по ключу будущего слота, если слот еще не добавлен в схему;
  - в `1. Разрешение атрибутов` выходной слот также можно задать как новый ключ, если его еще нет в выпадающем списке слотов сценария;
  - новый профиль всегда стартует как draft с источником данных "отключено" и не копирует endpoint-операцию из профиля-образца;
  - в `0. Слоты` при сохранении слота со способом "профиль разрешения" и пустым профилем автоматически создается draft-профиль для этого слота и слот сразу привязывается к нему;
  - draft-профиль обязан быть донастроен администратором перед переводом в полноценное исполнение.
- Каждый step с типом tool call ссылается на существующий tool, подключение и операцию.
- Каждый step использует только объявленные входные и выходные атрибуты.
- Clarification step содержит вопрос и список уточняемых атрибутов.
- History step содержит фильтры статусов, качества, периода и разрешенных полей.
- Fallback задан явно для каждого неоднозначного или неуспешного пути.
- Нельзя активировать слот, который ссылается на профиль, не выдающий именно этот слот как output.

Критерии выхода:

- В консоли администратора есть отдельный раздел "1. Разрешение атрибутов" с операциями создать, модифицировать и удалить.
- В разделе "1. Разрешение атрибутов" нет сводной таблицы профилей и JSON-редактора шагов; поведение профиля настраивается формовыми полями.
- В сценариях слот выбирает готовый профиль разрешения атрибута, а не технический источник данных.
- Для сценария сброса пароля можно описать pipeline поиска `login_name` через LLM extraction, AD lookup и уточняющий вопрос при неоднозначности.
- История заявок используется только через явно настроенный профиль.
- UI и backend не содержат hardcoded списка источников слота.
- Тестовый прогон сценария показывает шаги разрешения атрибутов и вопросы уточнения без реального вызова внешних систем.

### Этап 12.7: resolver-профиль разрешения атрибута

Цель: упростить модель `1. Разрешение атрибутов`, чтобы профиль был не mini-workflow с ручными переходами, а детерминированным resolver:

```text
признаки -> операция разрешения -> оценка результата операции -> матрица решений -> заполнить слот / уточнить / передать человеку
```

Поставки:

- JSON Schema профилей разрешения атрибутов содержит:
  - признаки `input_attributes`;
  - операцию разрешения `candidate_source`;
  - правила чтения ответа `result_policy`;
  - матрицу решений `decision_policy`;
  - правила уточнения `clarification_policy`;
  - правила передачи человеку `handoff_policy`.
- Из активной модели и UI убраны:
  - `steps`;
  - `resolution_mode`;
  - `attempt_scope`;
  - `intermediate_attributes`;
  - `ambiguity_policy`;
  - `operator_handoff_package`;
  - `on_success_step`, `on_failure_step`, `on_ambiguous_step`;
  - ручные условия вида `ad_candidates.count == 1`.
- Default-профиль сброса пароля:
  - извлекает признаки личности;
  - вызывает операцию разрешения `search_ad_users`;
  - читает тип результата, список `users`, целевое значение `login`, дополнительный выход `user_id`;
  - автоматически заполняет слот только при единственном результате и достаточном confidence;
  - при пустом или множественном результате задает уточняющий вопрос;
  - при ошибке источника или лимите попыток передает человеку.
- Backend dry-run возвращает `resolution_state`:
  - операцию разрешения;
  - сводку результата операции;
  - решение матрицы;
  - следующий вопрос;
  - пакет передачи человеку.
- Admin UI показывает форму без JSON-редактора и без workflow-шагов:
  - признаки для поиска;
  - операцию разрешения атрибута;
  - как оценивать результат операции;
  - матрицу решений;
  - уточнение и передачу человеку.
- Operator UI в шаге `1. Приём и нормализация` показывает оператору операцию разрешения, решение dry-run, вопрос и пакет передачи.

Архитектурные решения:

- LLM извлекает признаки и может помогать с текстом вопроса, но не управляет ветвлениями.
- Решение `заполнить / уточнить / передать` принимает Python-бэкенд по `decision_policy`.
- Формальный язык условий в UI не вводится; администратор выбирает действия для фиксированных случаев: результата нет, один результат, несколько результатов, ошибка источника, лимит попыток.
- ReAct-вызовы остаются runtime-адаптерами операции разрешения, а не языком описания workflow профиля.

Валидации перед активацией:

- `target_slot_id` входит в `output_slots`.
- `candidate_source` ссылается на существующий ReAct-вызов, endpoint и operation.
- `candidate_source.parameter_mapping` ссылается только на объявленные признаки, слоты, данные обращения, контекст, константы или секреты.
- `result_policy.output_mapping` заполняет только объявленные `output_slots`.
- `clarification_policy` содержит вопрос и уточняемые признаки.
- Пороги confidence идут от строгого автозаполнения к более мягкой передаче человеку.

Критерии выхода:

- Профиль `Поиск логина в AD по ФИО` описан через признаки, операцию разрешения, правила оценки результата и матрицу решений.
- Dry-run показывает решение resolver-профиля для недостающего слота.
- Админский UI не требует ручных условий и переходов.
- Операторский UI показывает, почему задан вопрос и какой шаг будет следующим.

### Этап 13: расширенное управление платформой

Цель: после завершения сценарной модели и профилей разрешения атрибутов превратить консоль администратора из обзорной консоли в управляемый контур изменения конфигурации AI-платформы.

Поставки:

- Процесс draft/validate/activate для:
  - ReAct-вызовов;
  - точек интеграции;
  - состояний рабочего процесса;
  - правил переходов;
  - промптов;
  - маршрутизации моделей;
  - каталога workflow n8n.
- Валидация по JSON Schema как обязательная проверка перед активацией.
- Регрессионные проверки перед активацией конфигурации промптов, ReAct-вызовов и workflow.
- Оценка промптов и качества поиска.
- Расширенные операции workflow n8n:
  - регистрация;
  - связь workflow с бизнес-сценарием;
  - конфигурация callback;
  - история запусков;
  - restart/cancel для поддерживаемых workflow.
- Маршрутизация нескольких моделей:
  - сценарий -> модель;
  - цепочки fallback;
  - лимиты запросов;
  - длина контекста;
  - настройки temperature.
- Интеграция с Vault.
- IP-ограничения.
- Поддержка MFA.
- Расширенная наблюдаемость:
  - LangGraph tracing;
  - структурированные логи;
  - расход токенов;
  - задержка;
  - SLA metrics;
  - аналитика ошибок интеграций.
- Процедура backup/restore для PostgreSQL и будущих persistent stores.

Архитектурные решения:

- Любая активация создает событие аудита и неизменяемую версию конфигурации.
- Регрессионные проверки могут блокировать активацию.
- Пустой подготовленный набор оценочных кейсов допускает bootstrap-активацию только со статусом gate `skipped`.
- Продуктовые действия по-прежнему проходят через Case API, Tool Registry и Execution Policy.
- n8n выполняет integrations, timers и polling, но не владеет business state кейса.
- Реальные n8n restart/cancel требуют n8n management API; до этого операции остаются guarded/unsupported.

Критерии выхода:

- Администратор может безопасно менять поведение AI, RAG, инструменты, workflow и model routing через версионированный процесс.
- Версии конфигурации можно откатить.
- Перед изменением production-поведения можно запустить evaluation/regression suite.
- Безопасность и наблюдаемость достаточны для пилотной эксплуатации.

### Этап 13.5: каналы взаимодействия и нейтральная эскалация

Цель: убрать из сценарной модели прямую привязку к уровням поддержки и вынести различия мессенджер-бота, сервисдеска и отладочного режима в расширяемый каталог каналов взаимодействия.

Поставки:

- Новый домен `interaction_channels` в Config Registry.
- JSON Schema `contracts/config/interaction-channels.schema.json`.
- Default-каналы:
  - `messenger_bot`: онлайн-интерактивный диалог с пользователем, короткие таймауты, незавершенное обсуждение создает черновик, эскалация зовет специалиста;
  - `service_desk`: офлайн-интерактивная заявка, ожидание привязано к SLA, эскалация создает наряд;
  - `debug`: текущий MVP-режим, вопросы идут оператору, эскалация останавливает сценарий с сообщением.
- Сценарий выбирает:
  - `default_channel_id`;
  - `allowed_channel_ids`.
- Admin UI:
  - `Сценарии обработки -> Каналы взаимодействия`;
  - создать, модифицировать, удалить канал;
  - настройка доставки вопроса, таймаутов, действия при отсутствии ответа, незавершенного обсуждения и эскалации;
  - блок "где используется" и запрет удаления канала, пока он выбран в сценариях.
- Operator UI:
  - отображает канал сценария;
  - показывает, куда должен уйти уточняющий вопрос;
  - показывает таймауты и действие эскалации в шаге `5. Решение и эскалация`.
- Маршруты классификации используют нейтральные значения:
  - `agent_with_confirmation`;
  - `human_review`;
  - `major_incident`.

Архитектурные решения:

- Сценарий не содержит таймеры канала и способ эскалации; он только выбирает готовый канал.
- Канал не владеет business state кейса; он описывает interaction policy и действие интеграции.
- Отладочный режим остается первым безопасным режимом MVP: эскалация не вызывает внешние системы.
- Будущее автоисполнение добавляется через канал, execution policy и запуск ReAct-вызова matrix без переписывания сценариев.

Критерии выхода:

- В UI и операторской панели нет продуктовых подписей, привязанных к конкретным уровням поддержки.
- Сценарий нельзя сохранить с неизвестным каналом.
- Канал нельзя удалить, если он используется в `default_channel_id` или `allowed_channel_ids`.
- Dry-run сценария возвращает `interaction_channel`, `question_delivery`, `waiting_policy` и `escalation_action`.
- Smoke-проверка подтверждает default-каналы, UI-маркеры и валидацию связей.

### Этап 13.6: профили действий канала для решения и эскалации

Цель: убрать из блока `5. Решение и эскалация` прямую настройку действий вроде "оповестить дежурных" и сделать канал взаимодействия единственным местом, где задается фактическая доставка эскалации.

Поставки:

- Расширение `interaction_channels`:
  - список `action_profiles`;
  - профиль содержит название, тип события, действие, optional tool, подключение, операцию и шаблон сообщения;
  - поддерживаемые типы событий: `standard_handoff`, `no_answer`, `major_incident`, `policy_blocked`, `debug_stop`;
  - поддерживаемое действие `notify_on_call`.
- UI блока `5. Решение и эскалация`:
  - поле "Оповещать дежурных" убрано;
  - порог Major Incident остается в блоке 5, потому что это условие принятия решения.
- UI `Каналы взаимодействия`:
  - настройка профилей действий канала карточками формы;
  - действие `оповестить дежурных` настраивается внутри профиля канала, а не в политике эскалации.
- Operator UI:
  - показывает вычисленные профили канала;
  - показывает действие профиля и optional tool/подключение/операция.

Архитектурные решения:

- Блок 5 отвечает за "когда и почему".
- Канал взаимодействия отвечает за "как и куда".
- Сценарий выбирает канал, канал исполняет профиль по `event_type`.
- Для `debug` профили останавливают сценарий с сообщением.
- Для `messenger_bot` Major Incident может оповестить дежурных.
- Для `service_desk` Major Incident создает срочный наряд дежурной группе.

Критерии выхода:

- `notify_on_call` встречается только как действие профиля канала.
- Политика эскалации не содержит UI-переключателя "Оповещать дежурных".
- Dry-run возвращает `channel_action_profiles`.

### Этап 13.7: автоматический выбор профилей канала

Цель: убрать из блока `5. Решение и эскалация` ручной выбор профилей канала. Профили уже принадлежат каналу, а сценарий уже выбирает канал, поэтому дополнительная связка в политике эскалации избыточна.

Поставки:

- Удаление пользовательского `channel_profile_mapping` из `escalation_policies`.
- Нормализация старых payload:
  - удалить `channel_profile_mapping`;
  - удалить legacy `major_incident.notify_on_call`.
- UI блока `5. Решение и эскалация` содержит только условия:
  - автозакрытие;
  - ожидание;
  - условия передачи;
  - Major Incident threshold;
  - пакет передачи;
  - шаблон уведомления пользователя.
- Runtime/dry-run выбирает профили по правилу:
  - `scenario.default_channel_id -> interaction_channel.action_profiles[event_type]`.
- Валидация каналов:
  - ровно один профиль для `standard_handoff`;
  - ровно один профиль для `no_answer`;
  - ровно один профиль для `major_incident`;
  - ровно один профиль для `policy_blocked`;
  - дубликаты обязательных `event_type` запрещены.

Критерии выхода:

- Блок `5` больше не содержит UI-блок `Профили канала`.
- `channel_profile_mapping` отсутствует в активных default-политиках эскалации.
- `notify_on_call` встречается только как действие профиля канала.
- Dry-run возвращает вычисленные `channel_action_profiles`.
- Smoke-проверка подтверждает автоматический выбор профилей по `event_type`.

### Этап 13.8: выбор условий и пакета передачи в блоке 5

Цель: убрать технические CSV-поля из блока `5. Решение и эскалация` и сделать настройку условий передачи и состава пакета понятной для администратора.

Поставки:

- Контракт `escalation_policies`:
  - `l2_conditions` удален;
  - `escalation_package` удален;
  - добавлен `handoff_conditions`;
  - добавлен `handoff_package`.
- Admin UI блока `5. Решение и эскалация`:
  - `Условия передачи` выводятся чекбоксами;
  - `Пакет передачи` выводится чекбоксами;
  - обязательные элементы пакета `Собранные слоты` и `Текст уведомления пользователя` включены и недоступны для отключения;
  - рядом с каждым пунктом есть короткое пояснение.
- Backend validation:
  - старые поля не принимаются;
  - `handoff_conditions` должен быть непустым;
  - `handoff_package` должен быть непустым;
  - `handoff_package` обязан содержать `slots` и `user_notification`;
  - Major Incident threshold остается не меньше 10.
- Operator UI:
  - показывает выбранные условия передачи;
  - показывает выбранный пакет передачи.

Архитектурные решения:

- Обратная совместимость старых полей не поддерживается.
- Блок `5` описывает "когда передавать" и "что передавать".
- Канал взаимодействия по-прежнему описывает "как и куда передавать".

Критерии выхода:

- В Admin UI нет ручного ввода `l2_conditions` и `escalation_package`.
- Active escalation policy не содержит старых полей.
- Payload со старыми полями не проходит валидацию.
- Нельзя сохранить пакет передачи без `slots` или `user_notification`.
- Smoke-проверка подтверждает новую модель блока `5`.

### Этап 13.9: чекбоксы ReAct-планирования

Цель: убрать технический ввод внутренних идентификаторов из блока `3. ReAct-планирование` и сделать настройку рамок ReAct-loop предметной для администратора.

Поставки:

- Контракт `orchestrator_policy`:
  - `allowed_tool_classes` удален;
  - добавлен `allowed_react_action_groups`;
  - `stop_conditions` переведен на предметные значения;
  - старые значения stop conditions не поддерживаются.
- Admin UI блока `3. ReAct-планирование`:
  - числовые поля `Максимум итераций` и `Ошибок ReAct-вызовов подряд до передачи` остаются;
  - `Разрешенные группы действий ReAct` выбираются чекбоксами;
  - `Стоп-условия` выбираются чекбоксами;
  - рядом с каждым пунктом есть короткое пояснение.
- Backend validation:
  - старое поле `allowed_tool_classes` не принимается;
  - `allowed_react_action_groups` должен быть непустым;
  - `stop_conditions` должен быть непустым;
  - `consecutive_tool_errors_to_escalate` не может быть больше `max_iterations`.
- Operator UI:
  - показывает группы действий ReAct русскими названиями;
  - показывает stop conditions русскими названиями.

Архитектурные решения:

- Блок `3` задает рамки планирования ReAct-loop.
- Блок `4` задает конкретные ReAct-вызовы ИИ, привязки операций и режим запуска.
- Блок `5` задает условия передачи и пакет контекста.
- Обратная совместимость старого `allowed_tool_classes` не поддерживается.

Критерии выхода:

- В Admin UI нет поля `Классы инструментов`.
- Active orchestrator policy не содержит `allowed_tool_classes`.
- Payload со старым `allowed_tool_classes` не проходит валидацию.
- Нельзя сохранить пустые `allowed_react_action_groups`.
- Smoke-проверка подтверждает новую модель блока `3`.

### Этап 13.17: справочник интеграций, ReAct-вызовов и привязок операций

Цель: убрать неоднозначный термин "инструмент" из пользовательской модели и разделить настройку исполнения на три понятных справочника внутри меню `Вызовы и интеграции`: `Интеграции`, `ReAct-вызовы ИИ` и `Привязка операций`.

Поставки:

- Контракт integration endpoint:
  - операция endpoint имеет `display_name` и `description`;
  - `operation_id` остается техническим ключом внутри подключения.
- Контракт proposed action:
  - `tool_name` больше не является hardcoded enum;
  - допустимость tool проверяется по активному `tool_catalog`.
- Admin UI:
  - подключения сгруппированы по adapter type;
  - подключение создается, модифицируется и удаляется через формы;
  - операции endpoint создаются, модифицируются и удаляются внутри карточки подключения;
  - ReAct-вызовы ИИ создаются, модифицируются и удаляются через формы;
  - привязки операций создаются, модифицируются и удаляются отдельным меню;
  - привязка операции выбирает ReAct-вызов ИИ, подключение и операцию выбранного подключения;
  - схемы параметров и результата ReAct-вызова редактируются как JSON Schema.
- Backend validation:
  - tool catalog проверяет ссылки на существующие endpoint и operation_id;
  - integration endpoint catalog проверяет активные tool bindings и n8n workflow;
  - удаление endpoint, operation, tool или binding блокируется, если активные домены продолжают ссылаться на объект.

Архитектурные решения:

- `Интеграции` владеет техническими деталями endpoint: adapter, URL, auth, path операции, timeout и mock output.
- `ReAct-вызовы ИИ` владеет бизнес-действием, схемами параметров/результата и политикой риска.
- Отдельной сущности профиля endpoint больше нет: есть техническое подключение `endpoint_id`, а операции редактируются внутри подключения.
- Дефолтные подключения укрупнены до `mock` и `n8n`; старые идентификаторы вида `mock.runbooks` и `n8n.identity` нормализуются только как legacy-вход.
- `Привязка операций` соединяет ReAct-вызов ИИ с одной текущей операцией endpoint; кнопка `Привязать` заменяет старую связь, кнопка `Отвязать` удаляет связь только для неиспользуемого ReAct-вызова.
- `4. ReAct-вызовы и матрица запуска` не содержит технический паспорт endpoint и только показывает текущую привязку операции.
- Источником истины для допустимых tools становится `tool_catalog`, а не схема AI output.
- Слово `tool` остается техническим именем домена в контрактах и API, но в UI показывается как `ReAct-вызов ИИ`.

Критерии выхода:

- В подразделах `Вызовы и интеграции -> Интеграции`, `Вызовы и интеграции -> ReAct-вызовы ИИ` и `Вызовы и интеграции -> Привязка операций` можно создать, изменить и удалить подключение, операцию, ReAct-вызов ИИ и текущую привязку операции.
- UI показывает `Где используется` и блокирует удаление используемых объектов.
- Контрактная проверка подтверждает ссылки tool -> endpoint -> operation_id и n8n workflow -> endpoint -> operation_id.

### Этап 13.18: разделение параметров вызова и операции endpoint

Цель: убрать перегрузку блока `4. ReAct-вызовы и матрица запуска` и разделить смысловые параметры вызова от технического payload операции подключения.

Поставки:

- Контракт operation endpoint:
  - операция содержит `request_schema` как JSON Schema технического входа adapter;
  - `mock_output` остается только тестовым ответом операции.
- Контракт tool binding:
  - binding содержит `parameter_mapping` по модели `параметр операции -> react:<param> / constant:<value> / secret:<env>`;
  - `tool.parameters_schema` остается смысловой схемой параметров вызова.
- Runtime:
  - `ToolRegistry` валидирует параметры вызова по `tool.parameters_schema`;
  - перед dispatch собирается `operation_parameters` по `parameter_mapping`;
  - `operation_parameters` валидируются по `operation.request_schema`;
  - n8n получает технический payload операции, а исходные параметры вызова передаются отдельно как `react_parameters`.
- Admin UI:
  - в `Интеграции` операция редактирует `Вход операции, JSON Schema` и `Тестовый ответ операции`;
  - в `Привязка операций` настраивается маппинг payload операции;
  - в `4. ReAct-вызовы и матрица запуска` блок переименован в `Параметры вызова`, необязательные параметры свернуты, технический payload операции не редактируется.
- Operator UI:
  - шаг 4 показывает параметры вызова отдельно от read-only подключения/операции.

Архитектурные решения:

- Сценарий не знает о технических параметрах adapter; он заполняет только параметры вызова.
- Операция подключения владеет техническим request contract.
- Привязка операции является единственным местом, где смысловые параметры вызова превращаются в payload endpoint.
- Старые активные конфигурации без `request_schema` и `parameter_mapping` нормализуются в безопасную схему object + mapping 1:1, если имена параметров совпадают.

Критерии выхода:

- Контрактная проверка подтверждает наличие `request_schema` у операций и закрытие обязательных параметров операции через `parameter_mapping`.
- Smoke-проверка dispatch падает с понятной ошибкой, если обязательный параметр operation request не закрыт маппингом.
- UI блока `4` не содержит текста "Маппинг параметров операции" и не редактирует технический payload endpoint.

### Этап 13.19: удаление универсальных ITSM-полей из payload операций

Цель: убрать слой "общих параметров на всякий случай" из ReAct-вызовов, endpoint-операций, заявки и RAG-фильтров.

Поставки:

- ReAct-вызовы используют только параметры, которые реально нужны действию: `target_ref`, `object_ref`, `query`, а для ранбуков - `runbook_code`, `user_login`, `account_type`, `device_name`, `app_name`, `error_text`.
- Endpoint-операции принимают тот же минимальный payload без универсальных полей маршрутизации.
- Матрица запусков маппит слоты сценария только в параметры выбранного ReAct-вызова.
- Имя ранбука как свободный текст не является полем заявки. Если endpoint требует выбор конкретного ранбука, используется детерминированный параметр вызова вроде `runbook_code`, который маппится в payload операции.
- Вход заявки больше не содержит отдельное поле среды, RAG-запросы не строят фильтры по нему.

Критерии выхода:

- Активные каталоги tools, integration endpoints и матрицы запусков не показывают универсальные поля.
- Smoke-проверки dispatch и approval для ранбуков продолжают работать через параметры вызова `runbook_code` и сценарные поля.
- UI оператора и администратора показывает параметры вызова и правила их заполнения, а не универсальный "целевой объект действия".

### Этап 13.20: детерминированные параметры вызова и маппинг endpoint

Цель: убрать неоднозначность между смысловыми параметрами сценария и техническими параметрами endpoint/runbook.

Поставки:

- В UI `Параметр ReAct-вызова` заменяется на `Параметр вызова`.
- В матрице запуска колонка `Значение` заменяется на правило заполнения: слот, поле кейса, контекст, константа или секрет.
- JSON Schema параметров вызова может задавать человекочитаемый `title`; UI показывает его рядом с техническим именем.
- Для ранбуков `target_ref` заменяется на явные параметры вызова: `runbook_code`, `user_login`, `account_type`, `device_name`, `app_name`, `error_text`.
- Endpoint-операция System Center принимает собственные параметры payload: `runbook_code`, `login`, `account_type`, `device_name`, `app_name`, `error_text`.
- `Привязка операций` выполняет детерминированный маппинг `параметр endpoint-операции -> параметр вызова / константа / секрет`.

Критерии выхода:

- В сценарной матрице видно, из какого слота или константы заполняется каждый параметр вызова.
- В привязке операции видно, какой параметр endpoint получает какой параметр вызова.
- `target_ref` больше не используется для запуска ранбуков.

### Этап 13.21: интерпретация результата операции в профилях разрешения атрибутов

Цель: убрать неоднозначность из `1. Разрешение атрибутов`: операция разрешения имеет входные параметры, а ее ответ оценивается отдельной детерминированной политикой результата.

Поставки:

- Контракт `attribute_resolution_profiles`:
  - `candidate_mapping` заменен на `result_policy`;
  - `result_policy.result_type` поддерживает `list` и `object`;
  - для `list` задаются `list_path`, `target_value_path`, опциональные `confidence_path`, `display_value_path`, `output_mapping`;
  - для `object` задаются `object_path`, `success_path`, `target_value_path`, опциональные `display_value_path`, `output_mapping`;
  - `decision_policy` использует случаи `empty_result`, `single_result`, `multiple_results`, `source_error`, `attempt_limit`.
- Backend:
  - dry-run считает количество результатов из `result_policy`, а не из отдельного `candidate_count`;
  - для object-ответа учитывает boolean-признак успешного поиска;
  - дополнительные выходные слоты заполняются только из `result_policy.output_mapping`;
  - источники входных параметров операции могут ссылаться на уже имеющиеся слоты; циклы не запрещаются на уровне конфигурации и ограничиваются лимитом попыток сценария.
- Admin UI:
  - блок `Источник кандидатов` был вынесен из интерфейса в пользу последующего блока `Обогащение контекста`;
  - `Параметр операции` переименован во `Входной параметр операции`;
  - `Источник` переименован в `Значение взять из`;
  - блок `Как читать кандидата` заменен на `Как оценивать результат операции`;
  - источники слотов показываются как `Слот: <название>`, а не как внутренние ссылки.
- Operator UI:
  - показывает тип результата и сводку ответа операции;
  - показывает решение dry-run через `empty_result / single_result / multiple_results`.

Критерии выхода:

- В UI профиля видно различие между входным параметром операции и полем ответа операции.
- В профиле больше нет ручного поля `Путь количества кандидатов`.
- Smoke-проверки подтверждают `result_policy`, новую матрицу решений и отображение операции разрешения.

### Этап 13.22: расширенная классификация маршрутов

Цель: заменить плоские `keywords` на управляемые правила классификации и выбирать маршрут по оценке всех активных маршрутов.

Поставки:

- Контракт `classification_routes`:
  - `rules.keywords` и `rules.negative_keywords` удалены без обратной совместимости;
  - `rules.rule_items` содержит правила с текстом признака, типом совпадения, влиянием, весом, обязательностью, блокировкой и пояснением;
  - типы совпадений: `phrase`, `word`, `contains`;
  - влияние: `positive` или `negative`.
- Backend:
  - dry-run оценивает все активные маршруты, а не только маршрут выбранного сценария;
  - позитивные правила повышают confidence;
  - негативные правила снижают confidence;
  - блокирующее негативное правило или отсутствующее обязательное позитивное правило обнуляет confidence маршрута;
  - возвращаются выбранный маршрут, confidence, уровень решения, top-N маршрутов и совпавшие правила.
- Admin UI:
  - простые поля `Ключевые слова` и `Негативные ключевые слова` убраны;
  - правила редактируются карточками с добавлением, изменением и удалением.
  - `До N категорий при низкой уверенности` трактуется как верхний предел подсказок: если маршрутов меньше, показываются все доступные, а удаление маршрутов не блокируется старым значением N.
- Operator UI:
  - шаг `2. Классификация и маршрутизация` показывает выбранный маршрут, совпадение с маршрутом сценария, позитивные/негативные признаки и top-N альтернатив.

Критерии выхода:

- В UI и контракте нет старых `keywords` / `negative_keywords`.
- Dry-run показывает top-N маршрутов и причины выбора.
- Payload со старыми полями не проходит JSON Schema.

### Этап 13.23: вкладочная панель оператора

Цель: разгрузить операторский интерфейс и дать оператору широкую рабочую область для сценарного прохождения заявки.

Поставки:

- Layout:
  - слева фиксируется рабочий контекст заявки: текст обращения, выбор сценария, режим тестового прогона, вопросы по слотам и введенные ответы;
  - справа отображается широкая область деталей с вкладками;
  - desktop layout использует две колонки: контекст заявки и детали;
  - на узких экранах контекст заявки идет сверху, вкладки остаются доступными горизонтально.
- Вкладки правой области:
  - `5 шагов`: сценарная карта и пять шагов оркестратора;
  - `Результат`: AI-решение, обратная связь и копируемый текст;
  - `Кейс`: статус кейса и timeline;
  - `Согласования`: action gates и решения оператора;
  - `База знаний`: статус RAG и ручное перестроение индекса;
  - `Трассировка`: RAG, инструменты и диагностический снимок.
- Operator UI:
  - переключение вкладок выполняется на клиенте без повторной загрузки данных;
  - сценарная работа не запускается автоматически при открытии страницы, вводе текста или смене настроек;
  - кнопка `Анализировать` фиксирует текст заявки, который используется следующими итерациями уточнения слотов;
  - текущие render-потоки и API остаются прежними;
  - диагностическая вкладка больше не называется `JSON` в интерфейсе.

Критерии выхода:

- Три крупных блока больше не сжаты в одной строке.
- Оператор может пройти сценарий по пяти шагам и переключаться к результату, кейсу, согласованиям, базе знаний и трассировке.
- Smoke-проверки фиксируют наличие вкладок, JS-состояния и адаптивного layout.

### Этап 13.24: упрощение профилей разрешения атрибутов

Цель: убрать из `1. Разрешение атрибутов` промежуточные технические блоки ручной оценки кандидатов и перейти к поддерживаемой схеме "входные слоты -> обогащение контекста ReAct-вызовами -> LLM-правило -> выходные слоты / клиент / оператор".

Поставки:

- Контракт `attribute_resolution_profiles`:
  - вместо `input_attributes`, `candidate_source`, `result_policy`, `decision_policy`, `clarification_policy`, `handoff_policy` используется новая структура;
  - `input_slots` задает слоты сценария, которые подаются на шаги обогащения и LLM-правило;
  - `enrichment_steps` задает последовательный список ReAct-вызовов, маппинг параметров из слотов, выходных слотов, констант, секретов и ранее полученных сущностей;
  - каждый шаг сохраняет результат как именованную сущность с описанными полями результата;
  - `output_slots_order` задает выходные слоты, порядок заполнения, обязательность, подсказку источника значения и fallback;
  - `llm_resolution_script` содержит редактируемый администратором скрипт выбора, заполнения и уточнения;
  - `human_resolution_policy` задает уточняемые слоты, пакет передачи человеку и действия при fallback.
- Backend:
  - нормализует старые активные версии в новую структуру при чтении;
  - валидирует новую структуру через JSON Schema и кросс-проверки связей с tools/endpoints/slots;
  - dry-run возвращает `enrichment_steps`, `enrichment_entities`, `output_slots_order`, `llm_resolution_script`, `human_resolution_policy` и `llm_decision`;
  - при выключенном LLM dry-run останавливается на понятном состоянии ожидания LLM-правила, а при включенном LLM использует детерминированную имитацию решения.
- Admin UI:
  - блоки `Признаки для поиска`, `Матрица решений` и `Как оценивать результат операции` убраны;
  - профиль редактируется через входные слоты, последовательные шаги обогащения контекста, поля именованных сущностей, выходные слоты с порядком, LLM-скрипт и правила передачи человеку;
  - уточняемые слоты и пакет передачи человеку выбираются из слотов сценария;
  - настройки результата ReAct-вызова редактируются формой, без JSON на интерфейсе.
- Operator UI:
  - шаг разрешения атрибутов показывает обогащение контекста, решение dry-run/LLM-правила, заполненные значения, выходные слоты, следующий вопрос и пакет передачи.

Критерии выхода:

- Активный payload профилей не требует legacy-полей `candidate_source`, `input_attributes`, `result_policy`, `decision_policy`, `clarification_policy`, `handoff_policy`.
- Администратор может создать/изменить профиль без ручного JSON и без свободного ввода слотов там, где есть сценарный контекст.
- Удаление слотов сценария не ломается из-за дополнительных выходных слотов профиля, если удаленный слот не используется текущей схемой.
- Smoke-проверки обновлены на новую модель профиля.

### Этап 13.36: последовательное обогащение контекста в профилях разрешения

Цель: заменить одиночную `Операцию разрешения атрибута` на простой последовательный список ReAct-вызовов, где результат каждого шага получает имя и может использоваться ниже по списку.

Поставки:

- Контракт `attribute_resolution_profiles` использует `enrichment_steps` как источник истины.
- В каждом шаге задаются название, ReAct-вызов, маппинг параметров, имя сущности результата, поля результата и поведение при ошибке.
- Маппинг параметров шага допускает ссылки `slot:`, `output:`, `entity:`, `constant:` и `secret:`.
- Backend валидирует, что `entity:` может ссылаться только на результаты предыдущих шагов, а ReAct-вызов существует и имеет привязку endpoint-операции.
- Operator UI показывает не одиночную операцию, а обогащение контекста профиля.

### Этап 13.37: компактный UX обогащения контекста

Цель: применить общий UX-подход из `docs/UI_UX_APPROACH.md` к блоку `1. Разрешение атрибутов -> Обогащение контекста`, чтобы администратор видел цепочку ReAct-вызовов как компактный сценарий, а не как набор больших форм.

Поставки:

- Верхний уровень блока `Обогащение контекста` становится компактной таблицей шагов: номер, название, ReAct-вызов, сущность результата, действие при ошибке, действия.
- Одновременно раскрывается только один редактор шага.
- Новый шаг раскрывается автоматически после добавления.
- Параметры ReAct-вызова отображаются плотной таблицей: параметр, тип/обязательность, источник значения.
- Источники параметров показываются человекочитаемо: слот, выходной слот, сущность предыдущего шага, константа или секрет.
- Контракт результата свернут по умолчанию в строку вида `Контракт результата: N полей`.
- При смене ReAct-вызова UI пересобирает параметры и поля результата по контракту выбранного вызова.
- Backend-контракт `enrichment_steps` не меняется.

Критерии выхода:

- В блоке не отображаются сразу большие формы всех шагов.
- Администратор видит порядок и смысл всей цепочки без длинного скролла.
- Сохранение продолжает писать тот же payload `enrichment_steps`.
- Smoke-проверки фиксируют компактный список, один раскрытый редактор и сохранение профиля.

### Этап 13.25: согласование контрактов ReAct-вызовов и endpoint-операций

Цель: убрать неявное согласование между виртуальным ReAct-вызовом и реальной endpoint-операцией. Реальный endpoint-контракт становится источником фактического запроса/ответа, а ReAct-вызов хранит виртуальный контракт для оркестратора. Между ними задается явный маппинг параметров и результата.

Поставки:

- Контракты:
  - у ReAct-вызовов добавлены `contract_version` и `contract_status`;
  - у endpoint-операций добавлены `response_schema`, `contract_version` и `contract_status`;
  - `endpoint_binding` содержит `parameter_mapping` для запроса и `result_mapping` для ответа;
  - `mock_output` endpoint-операции валидируется по `response_schema`.
- Backend:
  - валидирует совместимость типов `parameter_mapping` между `parameters_schema` ReAct-вызова и `request_schema` endpoint-операции;
  - валидирует, что обязательные поля `result_schema` ReAct-вызова заполняются через `result_mapping`;
  - проверяет существование путей в `response_schema` endpoint-операции и совместимость типов результата;
  - блокирует используемые ReAct-вызовы и endpoint-операции со статусом `broken`;
  - при dispatch сначала проверяет ответ endpoint по `response_schema`, затем приводит его к `result_schema` ReAct-вызова.
- Admin UI:
  - в `Инструменты и интеграции` редактируется схема ответа endpoint-операции и статус ее контракта;
  - в `ReAct-вызовах` редактируется версия и статус виртуального контракта;
  - в `Привязке операций` появился явный маппинг результата операции в результат ReAct-вызова и панель совместимости контрактов.

Архитектурные решения:

- Endpoint-операция описывает реальный технический контракт интеграции.
- ReAct-вызов описывает удобный для оркестратора виртуальный контракт.
- Сценарии и профили разрешения атрибутов ссылаются на ReAct-вызовы, а не на детали endpoint-контрактов.
- Если endpoint-контракт меняется несовместимо, ломается привязка операции, а не все сценарии молча.

Критерии выхода:

- Администратор видит и редактирует контракт endpoint-ответа без ручного поиска в JSON-файлах.
- Нельзя активировать привязку, которая не заполняет обязательный результат ReAct-вызова.
- Нельзя активировать endpoint-операцию с `mock_output`, не соответствующим `response_schema`.
- Используемые контракты со статусом `broken` не проходят validation.

### Этап 13.26: read-only граф оркестрации

Цель: дать администратору масштабируемую визуальную карту оркестратора/LangGraph-процесса без возможности редактирования графа напрямую.

Поставки:

- Backend:
  - read-only endpoint `GET /admin/orchestration-graph`;
  - режимы `view=base` и `view=scenario`;
  - для сценарного режима опциональный `scenario_id`;
  - ответ содержит `nodes`, `edges`, `layout`, `warnings`, `readonly`;
  - узлы содержат `id`, `title`, `step_number`, `type`, `status`, `description`, `config_refs`, `metrics`, `layout`, `readonly`;
  - ребра содержат `from`, `to`, `label`, `condition`, `type`.
- Admin UI:
  - раздел `Сценарии обработки -> Граф оркестрации`;
  - выбор сценария и режима отображения;
  - SVG-граф с узлами, переходами, условиями и read-only деталями узла;
  - масштабирование `+`, `-`, `100%`, `Вписать`;
  - перетаскивание полотна мышью и zoom колесом;
  - ссылки из узла в существующие разделы настройки.

Архитектурные решения:

- Граф не является новым источником истины.
- Редактирование остается в существующих сущностях: сценарии, слоты, разрешение атрибутов, классификация, ReAct-политика, матрица запусков, prompt pack, эскалация, каналы, integrations.
- На MVP граф строится из активной конфигурации, а не из ручного JSON.
- Позже можно добавить overlay фактического прогона по `execution_trace`.

Критерии выхода:

- Endpoint возвращает read-only граф для базового вида и выбранного сценария.
- Для `password_reset` отображаются шаги `0-5`.
- В UI есть масштабирование и перетаскивание полотна.
- В графе нет кнопок создания/удаления/сохранения узлов или ребер.

### Этап 13.27: разделение уточнения у клиента и эскалации оператору

Цель: убрать смешение двух разных процессов работы с человеком:

- `Уточнение у клиента` продолжает AI-сценарий после ответа клиента.
- `Эскалация оператору` завершает самостоятельную AI-обработку и передает оператору пакет контекста.

Поставки:

- Backend / dry-run:
  - `next_client_question` как явное поле вопроса клиенту;
  - `client_question` с delivery, waiting policy и семантикой `client_clarification`;
  - `awaiting_client_response` для состояния ожидания клиента;
  - `operator_escalation` с причиной, каналом, действием и семантикой `operator_escalation`;
  - `escalation_package` только когда реально требуется эскалация оператору;
  - старые поля `next_question`, `question_delivery`, `escalation_action` остаются техническими алиасами.
- Admin UI:
  - в `Каналах взаимодействия` настройки разделены на блоки `Уточнения у клиента` и `Эскалация оператору`;
  - в `1. Разрешение атрибутов` блок переименован в `Уточнение у клиента и эскалация оператору`;
  - уточняемые слоты явно относятся к клиентскому уточнению;
  - пакет handoff явно называется пакетом эскалации оператору;
  - в `5. Решение и эскалация` условия и пакет называются условиями/пакетом эскалации оператору.
- Operator UI:
  - вопрос по слотам отображается как `Нужно уточнение у клиента`;
  - в шаге 5 отдельно показаны `Ожидает клиента` и `Передано оператору`;
  - пакет передачи переименован в пакет эскалации.
- Граф оркестрации:
  - отдельная ветка `Ожидание ответа клиента -> возврат в slot filling`;
  - финальный узел `Эскалация оператору`;
  - ребра различают вопрос клиенту и эскалацию оператору.

Критерии выхода:

- Dry-run явно различает клиентское уточнение и операторскую эскалацию.
- UI не называет вопрос клиенту эскалацией.
- Эскалация всегда означает остановку AI-цикла и передачу оператору/группе через канал.
- Smoke-проверки фиксируют новую терминологию и поля dry-run.

### Этап 13.28: очистка активных вариантов меню администратора

Цель: убрать из пользовательского UI варианты, которые выглядят рабочими, но в текущем MVP являются каркасом или future-state.

Поставки:

- Admin UI:
  - в `1. Разрешение атрибутов` поле `Тип выполнения` переименовано в `Источник данных для разрешения`;
  - активные источники разрешения атрибута ограничены `endpoint-операцией чтения` и `отключено`;
  - при отключенном источнике скрываются выбор ReAct-вызова, маппинг параметров и описание результата операции;
  - в `0. Слоты` поле `Способ заполнения` переименовано в `Как получить значение слота`;
  - из активного выбора слотов убран неисполняемый режим `из данных обращения`;
  - в `4. ReAct-вызовы и матрица запуска` поле `Вид запуска` переименовано в `Режим исполнения сейчас`;
  - в матрице запуска убран активный вариант `согласование руководителя`;
  - UI больше не показывает целевой и текущий режимы запуска как разные управляемые сущности;
  - в `Интеграциях` активными adapter types оставлены `mock` и `n8n webhook`;
  - в `Каналах взаимодействия` действия фильтруются по режиму канала, а технические поля ReAct-вызова/endpoint/operation скрыты.
- Backend:
  - validation запрещает активные профили разрешения атрибутов с неисполняемыми источниками;
  - validation запрещает включенные endpoint adapters кроме `mock` и `n8n_webhook`;
  - validation запрещает `approver_approval` в матрице запуска до реализации отдельного approver flow;
  - validation требует синхронизации `execution_level` и `target_execution_level`;
  - validation проверяет соответствие действий канала его режиму.
- Конфигурация:
  - заготовка `queue.known_incidents` отключена в дефолтном каталоге;
  - активная конфигурация интеграций переведена на отключенные неисполняемые adapters;
  - активная матрица ReAct-вызовов синхронизирована по режиму запуска.

Архитектурные решения:

- Future-варианты остаются в плане развития и контрактах, но не предлагаются как рабочие действия администратора.
- История заявок и данные обращения для разрешения атрибутов будут возвращены в UI только после появления реального исполнения.
- Direct HTTP и queue adapters будут возвращены в активный выбор после реализации dispatcher-исполнения.
- Согласование руководителя остается будущим approver flow, а не вариантом текущего запуска в MVP.

Критерии выхода:

- Администратор не может случайно выбрать неисполняемый source/adapter/execution mode.
- Существующие future-заготовки видны как неисполняемые текущие значения только при открытии старых объектов.
- Новые активируемые конфиги не проходят validation, если в них включены неисполняемые варианты.
- Сценарий сброса пароля продолжает проходить dry-run с текущей матрицей запуска.

### Этап 13.29: визуальный редактор контрактов endpoint-операций

Цель: сделать настройку контрактов endpoint-операций управляемой через UI, чтобы администратор не синхронизировал вручную JSON Schema и mock-ответ.

Поставки:

- Admin UI:
  - в карточке операции endpoint добавлены блоки `Входные параметры операции`, `Поля ответа операции`, `Тестовый ответ mock`;
  - входные параметры и поля ответа редактируются строками формы;
  - из формы автоматически собираются `request_schema` и `response_schema`;
  - для `mock` тестовый ответ строится из полей ответа операции;
  - `JSON контрольная точка` остается доступной в экспертном режиме;
  - кнопка `Применить JSON в форму` переносит поддерживаемую часть JSON Schema обратно в UI.
- Привязка операций:
  - блок `Входные параметры endpoint` показывает все параметры endpoint-операции и задает, какие из них видит ReAct-вызов;
  - блок `Поля ответа, доступные ReAct` выбирает поля endpoint-ответа, которые возвращаются в результат ReAct;
  - при сохранении UI пересобирает `parameters_schema`, `result_schema`, `parameter_mapping` и `result_mapping`;
  - при смене endpoint-операции UI не возвращается к сохраненной привязке, а сразу перестраивает видимый контракт по выбранной операции.
- Validation:
  - перед сохранением UI проверяет обязательные поля mock-ответа;
  - UI проверяет типы простых значений mock-ответа;
  - backend продолжает финальную проверку итогового JSON через `jsonschema`.
- Документация:
  - добавлен документ `docs/STAGE_13_29.md`.

Архитектурные решения:

- Backend-контракты не меняются: источником активации остается итоговый JSON.
- Пользовательский источник редактирования в UI - форма, а JSON является контрольной точкой.
- Сложные JSON Schema остаются экспертным режимом; MVP-визуальный редактор поддерживает простые object schemas.

Критерии выхода:

- Администратор может добавить, изменить и удалить входные параметры операции без ручного JSON.
- Администратор может добавить, изменить и удалить поля ответа операции без ручного JSON.
- Для mock endpoint тестовый ответ собирается из формы и проверяется по типам.
- Smoke-проверка фиксирует новые UI-блоки.

### Этап 13.30: видимый контракт ReAct поверх endpoint-операции

Цель: убрать смешение слотов сценария и технического endpoint-маппинга в разделе `Привязка операций`.

Поставки:

- `Привязка операций` больше не показывает список слотов и не создает впечатление сценарной настройки.
- Блок `Входные параметры endpoint` выводит все параметры `request_schema`, включая необязательные.
- Для каждого входного параметра endpoint выбирается источник: `параметр ReAct-вызова`, `константа`, `секрет` или `не использовать`.
- Параметры с источником `параметр ReAct-вызова` формируют `parameters_schema` ReAct-вызова.
- Блок `Поля ответа, доступные ReAct` выводит поля `response_schema` и позволяет выбрать любой набор для `result_schema`.
- При сохранении `Привязать` UI пересобирает `parameters_schema`, `result_schema`, `parameter_mapping` и `result_mapping`.
- Переключатель `Синхронизировать контракт ReAct` удален как лишний слой.
- Документация: добавлен документ `docs/STAGE_13_30.md`.

Архитектурные решения:

- Endpoint-операция остается техническим контрактом интеграции.
- ReAct-вызов остается видимым контрактом оркестратора.
- Слоты сценария заполняют параметры ReAct только в матрице запуска или профиле разрешения атрибута.
- `Привязка операций` отвечает только за границу `endpoint contract -> ReAct visible contract`.

Критерии выхода:

- Администратор видит все параметры endpoint-операции, а не только обязательные.
- Администратор может открыть ReAct только нужные входные параметры и поля ответа.
- Required endpoint и required ReAct управляются отдельно.
- Smoke-проверки фиксируют новую терминологию UI.

### Этап 13.31: создание ReAct-вызова из привязки операций

Цель: убрать ручное копирование одноименных параметров при создании нового ReAct-вызова поверх endpoint-операции.

Поставки:

- В разделе `Привязка операций` добавлен режим `Создать и привязать ReAct-вызов ИИ`.
- Администратор выбирает endpoint-операцию и сразу задает имя, тип действия и описание нового ReAct-вызова.
- Видимые входные параметры и поля ответа ReAct редактируются в тех же блоках `Входные параметры endpoint` и `Поля ответа, доступные ReAct`.
- При сохранении UI создает новый объект `tools` с `parameters_schema`, `result_schema`, `endpoint_bindings`, `parameter_mapping` и `result_mapping`.
- После создания UI переключается на созданный ReAct-вызов в режиме привязки существующего.
- Документация: добавлен документ `docs/STAGE_13_31.md`.

Критерии выхода:

- Создание нового ReAct-вызова не требует перехода в отдельный раздел `ReAct-вызовы ИИ`.
- Одноименные параметры endpoint и ReAct заполняются автоматически.
- Существующий режим привязки уже созданного ReAct-вызова сохранен.

### Этап 13.38: ReAct-автозаполнение слотов

Цель: дать администратору детерминированный способ заполнения одного или нескольких слотов из read-only ReAct-вызова, включая создание недостающих слотов из полей результата.

Поставки:

- Новый домен Config Registry `slot_autofill_profiles`.
- Новый способ заполнения слота `slot_autofill`.
- Admin UI:
  - добавлен раздел `Сценарии обработки -> 0.1 Автозаполнение слотов`;
  - профиль выбирает схему слотов и read-only ReAct-вызов;
  - исполняемость профиля задается только полем `Статус`: запускаются только `active`, `draft` и `disabled` не участвуют в автозаполнении;
  - входные параметры ReAct-вызова маппятся из слотов, данных обращения, контекста, констант или секретов;
  - поля результата можно игнорировать, привязать к существующему слоту или превратить в новый слот схемы;
  - при создании нового слота UI сначала активирует обновленную схему слотов, затем активирует профиль автозаполнения.
- Backend:
  - валидирует, что профиль использует существующую схему слотов, read-only ReAct-вызов и существующие целевые слоты;
  - проверяет обязательные параметры ReAct-вызова;
  - не дает удалить слот, если активный профиль автозаполнения продолжает ссылаться на него;
  - в dry-run выполняет mock/read-only автозаполнение до профилей разрешения атрибутов.
- Operator UI:
  - в первом шаге показывает строки автозаполнения слотов;
  - в таблице слотов отображает значения, заполненные через ReAct-автозаполнение.

Архитектурные решения:

- Runtime не создает слоты сам. Слот появляется только как результат draft-настройки администратора.
- ReAct-вызов может вернуть больше данных, чем нужно сценарию; активный профиль явно выбирает, какие поля становятся слотами.
- `slot_autofill_profiles` отличается от `attribute_resolution_profiles`: автозаполнение не запускает LLM-правило разрешения неоднозначности, а принимает только детерминированный результат по политике профиля.

Критерии выхода:

- Администратор может создать профиль автозаполнения и породить новый слот из поля результата ReAct-вызова.
- Конфигурация не активируется, если целевой слот отсутствует или поле результата не входит в контракт ReAct-вызова.
- Операторский dry-run показывает, какие слоты заполнены одним ReAct-вызовом.

### Этап 13.39: обязательность поля результата в профиле автозаполнения

Цель: разделить обязательность поля в общем контракте ReAct-вызова и обязательность поля для конкретного профиля автозаполнения.

Поставки:

- `slot_autofill_profiles.output_mapping[].required_for_success` сохраняется как настройка профиля, а не как отражение `required` в JSON Schema.
- Admin UI:
  - подпись изменена на `Поле результата обязательно для профиля`;
  - добавлен подстрочник о том, что профиль считается неуспешным, если выбранное значение не вернулось;
  - флаг доступен для любого поля результата, включая необязательные в контракте ReAct-вызова.
- Backend dry-run:
  - проверяет значения по всем строкам `output_mapping`;
  - если обязательное для профиля поле отсутствует, возвращает статус `missing_required_result_field`;
  - возвращает `missing_required_result_fields` и `missing_required_slots`;
  - найденные необязательные значения остаются в `output_values`, но профиль не считается успешным.
- Operator UI:
  - показывает статус отсутствующего обязательного поля результата;
  - выводит список полей результата и целевых слотов, из-за которых профиль не успешен.

Архитектурные решения:

- Контракт ReAct-вызова описывает, что вызов может вернуть.
- Профиль автозаполнения описывает, что нужно именно этому сценарию для успешного заполнения.
- Обязательность целевого слота в схеме слотов остается отдельной настройкой и не смешивается с обязательностью поля результата.

Критерии выхода:

- Администратор может сделать необязательное в контракте поле обязательным для профиля автозаполнения.
- Dry-run не считает профиль успешным, если это поле не вернулось.
- В операторском UI видно, какого поля результата не хватило.

### Этап 14: переоценка реестра промптов как библиотеки шаблонов

Цель: если система развивается дальше одного сценарного prompt pack, переосмыслить низкоуровневый домен `prompts` не как отдельный раздел настройки поведения агента, а как библиотеку переиспользуемых шаблонов промптов.

Поставки:

- Решение, нужен ли отдельный каталог шаблонов поверх `prompt_packs`.
- Модель "шаблон промпта -> где используется":
  - prompt pack;
  - сценарий;
  - тип блока system prompt;
  - evaluation/regression set.
- CRUD для шаблонов только при подтвержденной потребности переиспользования.
- Versioning шаблонов и безопасный rollback.
- Проверка, что изменение шаблона не меняет поведение сценариев без явного draft/validate/regression/activate.
- UI-название "Библиотека шаблонов промптов", если раздел будет возвращен в интерфейс.

Архитектурные решения:

- Основной UI настройки поведения AI остается в `Сценарии обработки -> 6. Промпты`.
- Библиотека шаблонов не должна конкурировать с prompt pack и не должна быть вторым местом изменения сценарного system prompt.
- Шаблон может быть только источником переиспользуемого текста или фрагмента, а фактическая активная конфигурация сценария фиксируется в prompt pack.

Критерии выхода:

- Принято решение: удалить низкоуровневый `prompts` из пользовательской плоскости окончательно или вернуть его как библиотеку шаблонов.
- Если библиотека нужна, администратор видит связи "где используется" и не может удалить шаблон, пока он используется.
- Если библиотека не нужна, сценарная модель остается единственным UI для промптов.
