# Этап 13.25. Согласование контрактов ReAct-вызовов и endpoint-операций

## Цель

Разделить два уровня контракта:

- endpoint-операция описывает реальный технический запрос и ответ интеграции;
- ReAct-вызов описывает виртуальный контракт, который видит оркестратор и LLM.

Связь между ними теперь задается явно через маппинг запроса и результата.

## Что изменено

- У ReAct-вызовов добавлены `contract_version` и `contract_status`.
- У endpoint-операций добавлены `response_schema`, `contract_version` и `contract_status`.
- В привязке endpoint-операции к ReAct-вызову используется:
  - `parameter_mapping` для входных параметров endpoint;
  - `result_mapping` для преобразования ответа endpoint в результат ReAct-вызова.
- Admin UI показывает схему ответа endpoint-операции, статус контракта и совместимость привязки.
- Dispatcher проверяет сырой ответ endpoint по `response_schema`, затем применяет `result_mapping` и валидирует итог по `result_schema` ReAct-вызова.

## Валидация

- Обязательные параметры `request_schema` endpoint-операции должны быть заполнены через `parameter_mapping`.
- Типы входных параметров ReAct-вызова и endpoint-операции должны быть совместимы.
- Обязательные поля `result_schema` ReAct-вызова должны быть заполнены через `result_mapping`.
- Пути `result_mapping` должны существовать в `response_schema` endpoint-операции.
- Типы ответа endpoint и результата ReAct-вызова должны быть совместимы.
- `mock_output` endpoint-операции должен соответствовать `response_schema`.
- Используемые ReAct-вызовы и endpoint-операции со статусом `broken` не проходят validation.

## Проверки

- `node --check apps/admin-ui/static/app.js`
- `node --check apps/operator-ui/static/app.js`
- `.venv/bin/python -m compileall -q apps/orchestrator`
- `scripts/stage12_5-smoke.sh`
