# Этап 13.24. Упрощение профилей разрешения атрибутов

## Цель

Раздел `Сценарии обработки -> 1. Разрешение атрибутов` переведен с технической схемы кандидатов на поддерживаемую модель:

`входные слоты -> обогащение контекста ReAct-вызовами -> LLM-скрипт -> выходные слоты / клиент / оператор`.

## Что изменено

- Контракт `attribute_resolution_profiles` теперь хранит:
  - `input_slots`;
  - `enrichment_steps`;
  - `output_slots_order`;
  - `llm_resolution_script`;
  - `human_resolution_policy`.
- Старые поля `input_attributes`, `candidate_source`, `result_policy`, `decision_policy`, `clarification_policy`, `handoff_policy`, `output_slots` больше не являются целевой моделью.
- Backend нормализует старые активные версии при чтении, но новые сохранения выполняются только в новой структуре.
- Admin UI убрал блоки `Признаки для поиска`, `Матрица решений`, `Как оценивать результат операции`.
- В профиле теперь редактируются входные слоты, последовательные ReAct-вызовы обогащения, поля именованных сущностей результата, выходные слоты с порядком, LLM-скрипт и правила уточнения/передачи человеку.
- Operator UI показывает решение LLM-правила, заполненные значения и выходные слоты профиля.

## Валидация

- `enrichment_steps[].parameter_mapping` может ссылаться на входные слоты, выходные слоты, константы, секреты и сущности предыдущих шагов.
- `target_slot_id` должен входить в `output_slots_order`.
- `clarification_slots` и `handoff_package` должны ссылаться на объявленные входные или выходные слоты профиля.
- ReAct-вызов проверяется по существующему каталогу и должен иметь привязку endpoint-операции.
- `llm_resolution_script.response_contract` фиксирует ожидаемые поля ответа модели.

## Проверки

- `node --check apps/admin-ui/static/app.js`
- `node --check apps/operator-ui/static/app.js`
- `.venv/bin/python -m compileall -q apps/orchestrator`
- `scripts/stage12_5-smoke.sh`
