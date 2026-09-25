# modes.md

В `README.md` добавил раздел «Как проверить, что сервис жив» (между таблицей команд и разделом «API`).
Описывает 5 способов проверки живости после `make up`: HTTP `GET /health`, `make ps` (статус контейнеров, в т.ч. healthcheck у `db`), `make logs`, `make test`, `docker compose exec backend curl .../health`.
Основано на `Makefile` (`up`, `ps`, `logs`, `test`) и `docker-compose.yml` (healthcheck у `db` через `mysqladmin ping`, `depends_on: service_healthy` у `backend`, эндпоинт `/health`).

Code - Выполняет действия в рамках своих permission, ask - отвечает на вопросы в режиме read only, debug - поиск и исправление багов, Оркестратор - коорлинирует всю задачу целиком делегируя разные части специальным агентам в параллельном режиме, Plan - планировщик - может только изменять файлы плана, все другие изменения файловой системы запрещены