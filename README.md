# PulseCourier

Единая платформа уведомлений (email/SMS/push/Slack/Telegram) с шаблонами, резервным провайдингом, очередями, ретраями, идемпотентностью, вебхуками и планированием отправок.

## Возможности
- Шаблоны (Jinja2)
- Провайдеры и резервный обход (есть консольный заглушечный провайдер)
- Очереди (Celery) и Redis
- Идемпотентность (Redis setnx)
- Наблюдаемость: Prometheus, Sentry, OpenTelemetry
- FastAPI с документацией OpenAPI

## Быстрый старт (Docker)
```bash
# Сборка и запуск
docker compose up --build
# Открыть документацию API
open http://localhost:8000/docs
```

## Локальная среда
Скопируйте `.env.example` в `.env` и поменяйте значения при необходимости.

## API
- GET `/api/v1/health`
- POST `/api/v1/notifications/send` с телом в формате схемы `Message` (см. `app/schemas/message.py`)

## Разработка
```bash
pip install -r requirements.txt
pip install -r requirements-dev.txt  # инструменты разработки: ruff, mypy, black
pre-commit install
pytest -q
uvicorn app.main:app --reload
```

### Быстрая проверка качества (QA)
- Windows (PowerShell):
```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File scripts/qa.ps1
```
- Linux/macOS (Bash):
```bash
bash scripts/qa.sh
```
Примечание: убедитесь, что dev-зависимости установлены (`pip install -r requirements-dev.txt`).

### Релиз одним скриптом (Windows)
Для полного прогона проверок, сборки/запуска docker-compose и публикации в GitHub:

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File scripts/release.ps1 -Remote "https://github.com/R0D10Nq/pulse-courier.git"
```

Параметры:
- `-Remote` — URL вашего удалённого репозитория (по умолчанию указан пример).
- `-SkipDocker` — если нужно пропустить шаг с docker-compose.
- `-SkipGit` — пропустить шаги git commit/push (для ручного коммита).

Примечание: если `origin` уже настроен, скрипт его не изменяет; чтобы явно задать/поменять — используйте `-Remote`.

Скрипт выполнит: pip install (prod/dev), Ruff fix, Black форматирование и проверку, mypy, pytest, `docker compose up -d --build` с health‑пробой `/api/v1/health`, затем сделает `git commit` и `git push main` (если не указан `-SkipGit`).

## Архитектура
- `app/main.py` — приложение FastAPI
- `app/api/routes.py` — эндпоинты
- `app/services/notifications.py` — постановка задач в очередь + идемпотентность
- `app/worker/` — Celery-приложение и задачи
- `app/providers/` — абстракция провайдеров + консольный провайдер
- `app/core/` — конфиг, логирование, наблюдаемость, redis, шаблоны
- `app/db/` — асинхронная сессия SQLAlchemy (Alembic будет добавлен позже)

**Архитектурные решения:**
- **Multi-provider notification system** (email/SMS/push/Slack/Telegram)
- **Fallback/retry mechanisms** с экспоненциальным backoff
- **Template engine** с персонализацией
- **Queue-based processing** с dead letter queues
- **Webhook callbacks** для delivery confirmations
- **Rate limiting** и **idempotency**

## Дорожная карта
- Реальные провайдеры: SES, SendGrid, Twilio, FCM, Slack, Telegram
- Ретраи/бэкофф и DLQ
- Вебхуки статусов доставки
- Планирование (ETAs/Beat)
- Миграции Alembic, фикстуры
- Ограничение скорости (rate limiting) на тенанта
- Экспортёры OpenTelemetry + дашборды
- Нагрузочные тесты k6/Locust

## Безопасность и качество
- Валидация входных данных через Pydantic
- Линтеры: Ruff, Black; Типизация: mypy; хуки pre-commit
- SAST: планируется добавить bandit/safety

## Чек-лист к релизу
- [ ] CI зелёный: Ruff, Black, mypy, pytest (`.github/workflows/ci.yml`)
- [ ] Файл окружения заполнен: `.env` (см. образец `.env.example`)
- [ ] Включить Celery в проде: `CELERY_TASK_ALWAYS_EAGER=false`
- [ ] (Опционально) Лимиты: `RATE_LIMITER_ENABLED=true`, настроен Redis
- [ ] (Опционально) Наблюдаемость: заданы `SENTRY_DSN`, `OTEL_EXPORTER_OTLP_ENDPOINT`
- [ ] Docker: `docker compose up --build` поднимает api/worker/redis/postgres/minio
- [ ] Smoke: `GET /api/v1/health` и `POST /api/v1/notifications/send` (канал `console`)

## Лицензия
MIT
