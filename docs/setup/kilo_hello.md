# kilo_hello.md

1) Сервис: учебный сервис **предварительной оценки заявки на заём под ПТС** — принимает заявку (VIN, год, пробег, оценочная стоимость, сумма, срок), считает LTV и возвращает решение `approve` / `review` / `reject` (см. `README.md` и `AGENTS.md` в корне `carmoney-lab/`).
2) Команды из `Makefile`: `make help`, `make up` (`docker compose up -d --build`), `make down`, `make ps`, `make logs`, `make install` (`composer install`), `make test` (PHPUnit локально или в контейнере `backend`), `make lint` (`php -l` по `backend/` и `tests/`), `make seed` (`mysql -ulab -plab carmoney_lab < db/seed.sql`). В `docker-compose.yml`: сервисы `backend` (PHP 8.3 + Slim, `php -S 0.0.0.0:8080`, порт `${APP_PORT:-8080}:8080`) и `db` (`mysql:8.0`, БД `carmoney_lab`, init-скрипты `db/schema.sql` и `db/seed.sql`, healthcheck `mysqladmin ping`), health: `GET /health` на `http://localhost:8080/`.
3) Решение `approve` / `review` / `reject` считается в папке `backend/src/Domain/` (класс `DecisionEngine::decide()` по порогам LTV из `backend/config/rules.php`; пороги задаёт `AssessmentService`).

модель: training-2026-09-minimax-m3
