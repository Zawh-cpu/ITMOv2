# Integration-проверки

Объект: связка `POST /api/reviews` (`app/api.py`), `ReviewService`, подставная `LLM` и `GET /health`. Допущения: pytest и `fastapi.testclient.TestClient`; `app.dependencies.review_service` неизвестен, поэтому в тестах подменяем `app.api.review_service` на `ReviewService(fake_llm)`. Запуск не выполнялся (кода сервиса в репозитории нет).

| Связь компонентов | Что может сломаться | Как воспроизводим | Ожидаемый результат | Подтверждение |
|---|---|---|---|---|
| I1. API -> `ReviewService` -> fake LLM | Ответ не доходит до клиента | `POST /api/reviews` с `{"diff": "x"}`, fake отвечает "ok" | 200 и тело `{"comment": "ok"}` | TestClient, проверка кода и тела |
| I2. Тело без поля `diff` | `KeyError` даёт необработанную ошибку | `POST /api/reviews` с `{}` (`curl -s -o /dev/null -w "%{http_code}" -X POST http://<host>:<port>/api/reviews -H "Content-Type: application/json" -d '{}'`) | Текущее поведение по diff: необработанное исключение (500). TO BE: контролируемая ошибка 4xx (допущение) | TestClient с `raise_server_exceptions=False` или curl; сравнить код |
| I3. Тело не объект (список `[]`) | Валидация типа `payload: dict` | `POST /api/reviews` с `[]` | Ожидание: 422 от FastAPI, так как тип параметра `dict` (проверить запуском) | TestClient |
| I4. Сбой LLM внутри запроса | Исключение доходит до клиента без контролируемого ответа | fake бросает `RuntimeError`, `POST` с `{"diff": "x"}` | Текущее поведение по diff: 500. TO BE: контролируемая ошибка (допущение) | TestClient с `raise_server_exceptions=False` |
| I5. Diff в промпте | Diff теряется или искажается по пути API -> сервис -> LLM | `POST` с многострочным diff, где есть `+`, `-` и `\n` | `fake.last_prompt` содержит diff без изменений | TestClient, сравнение строки |
| I6. Существующий `GET /health` не сломан | Регрессия из-за нового маршрута | `GET /health` | 200 и `{"status": "ok"}` | TestClient |
| I7. Реальная LLM (вне CI) | Формат ответа, таймауты, лимиты | `POST` с `TRAINING_PR.diff` при настроенном доступе; допущение: параметры доступа вне задачи | Ответ в трёх блоках, у рисков цитаты (проверка по [`tests_e2e.md`](tests_e2e.md)) | Ручной запуск, запись в `prompts.md` |

## Как использовали AI

- Строка в [`prompts.md`](prompts.md): строка 12 (P1-04).
- Что я проверил и какие исправления поручил: на момент записи я проверки не оценил; поведение FastAPI для I3 не проверялось запуском и помечено ожиданием.
