# Code map: как считается решение approve / review / reject

Источники: `backend/src/Domain/`, `backend/config/rules.php`, `backend/src/Http/ApplicationController.php`, `backend/src/AppFactory.php`.

## Цепочка вызовов (файл → функция → порядок)

1. **HTTP-слой**
   - `backend/src/Http/ApplicationController.php::create()` (POST `/api/applications`) и `::ltv()` (POST `/api/ltv`) — читают `getParsedBody()`, вызывают `AssessmentService::assess($payload)`. На `ValidationException` отдают 422 с массивом ошибок.

2. **Оркестратор**
   - `backend/src/Domain/AssessmentService.php::assess($payload)`:
     1. `$input = $this->validator->validate($payload);` — нормализация и валидация всех полей.
     2. `$ltv = $this->ltvCalculator->calculate($input['requested_amount'], $input['market_value']);` — расчёт LTV в процентах.
     3. `$decision = $this->decisionEngine->decide($ltv);` — единственное место, где формируется `approve` / `review` / `reject`.
     4. Сборка ответа: `vehicle_age`, `ltv`, `decision`, `approved_limit` (равен `requested_amount` при `approve`, иначе `0`), плюс нормализованный `input`.

3. **Валидация и нормализация**
   - `backend/src/Domain/ApplicationValidator.php::validate($payload)`:
     - VIN → `VinValidator::isValid()`.
     - Год → проверка через `VehicleAge::inYears()` на `min_year` / будущее / `max_age_years`.
     - Пробег → проверка диапазона `[0, max_mileage_km]` (см. ниже).
     - Сумма, срок — проверки по `rules.amount` / `rules.term`.
     - На любой ошибке — `throw new ValidationException($errors);`.
     - На успехе возвращает массив: `vin, year, mileage, market_value, requested_amount, term_months`.

4. **Расчёт LTV**
   - `backend/src/Domain/LtvCalculator.php::calculate($requestedAmount, $marketValue)` — `round($requested / $market * 100, 2)`. Бросает `InvalidArgumentException` при неположительных значениях.

5. **Решение**
   - `backend/src/Domain/DecisionEngine.php::decide($ltv)`:
     - `ltv < approve_max` (60.0) → `approve`
     - `approve_max <= ltv <= review_max` (85.0) → `review`
     - `ltv > review_max` → `reject`
   - Пороги берутся из `backend/config/rules.php` (`rules.ltv.approve_max`, `rules.ltv.review_max`).

6. **DI и роутинг**
   - `backend/src/AppFactory.php::create()` собирает цепочку и регистрирует роуты `/health`, `POST /api/ltv`, `POST /api/applications`, `GET /api/applications`, `GET /api/applications/{id}`.

```
ApplicationController::create/ltv
  └── AssessmentService::assess
        ├── ApplicationValidator::validate        ──► ValidationException → 422
        │     ├── VinValidator::isValid
        │     └── VehicleAge::inYears
        ├── LtvCalculator::calculate
        └── DecisionEngine::decide                ──► approve / review / reject
```

## Что уже проверяется про пробег

- **Конфиг**: в `backend/config/rules.php` есть единственный параметр пробега — `'vehicle' => [..., 'max_mileage_km' => 500000]` (строка 23). Используется только как верхняя граница диапазона.
- **Валидация**: `backend/src/Domain/ApplicationValidator.php:43–46`:
  ```php
  $mileage = (int) ($payload['mileage'] ?? -1);
  if ($mileage < 0 || $mileage > $this->rules['vehicle']['max_mileage_km']) {
      $errors['mileage'] = sprintf('Пробег от 0 до %d км', $this->rules['vehicle']['max_mileage_km']);
  }
  ```
  Пробег вне `[0, 500000]` → `ValidationException` → 422, решения по заявке нет.
- **Нормализация**: `mileage` (int) попадает в возвращаемый массив под ключом `'mileage'` (ApplicationValidator.php:78) и доступен в `AssessmentService::assess()` как `$input['mileage']`.
- **Решение**: `DecisionEngine::decide()` **не получает** пробег и **не учитывает** его. `approve` / `review` / `reject` сейчас зависит исключительно от LTV.
- **Лимит**: `approved_limit` тоже от пробега не зависит.

## Точка вставки правила «пробег ≤ 400 000 км, иначе review»

Архитектурно правильное место — `DecisionEngine`: вся логика `approve` / `review` / `reject` сосредоточена там и под неё подведены пороги из `rules.php`. Размещение в `AssessmentService` размазало бы правила по двум классам.

Что нужно изменить:

1. **`backend/config/rules.php`** — добавить отдельный порог для решения (текущий `max_mileage_km` = 500000 — это валидационный «потолок», его не трогаем):
   ```php
   'vehicle' => [
       ...
       'max_mileage_km'   => 500000,
       'review_mileage_km' => 400000,
   ],
   ```

2. **`backend/src/Domain/DecisionEngine.php`**:
   - Конструктор сейчас принимает `array{approve_max:float, review_max:float}` — расширить до `array{approve_max:float, review_max:float, review_mileage_km?:int}` и сохранить новое поле.
   - Сигнатура `decide(float $ltv): string` сейчас пробег не принимает — изменить на `decide(float $ltv, int $mileage): string`. Это ломает сигнатуру: затронуты `AssessmentService` и unit-тесты `DecisionEngine`.
   - В начале метода, до ветки по LTV, добавить:
     ```php
     if ($mileage > $this->reviewMileageKm) {
         return self::REVIEW;
     }
     ```
   Семантика из описания правила — «иначе review» — даёт именно это: пробег выше порога всегда понижает до `review`, даже если LTV был бы `approve`.

3. **`backend/src/Domain/AssessmentService.php`** — обновить вызов:
   ```php
   $decision = $this->decisionEngine->decide($ltv, $input['mileage']);
   ```
   Вход `mileage` уже доступен через `$input['mileage']` — дополнительных источников не требуется.

4. **`backend/src/AppFactory.php`** — в сборке `DecisionEngine` прокинуть новый ключ из `$rules`:
   ```php
   new DecisionEngine($rules['ltv'] + ['review_mileage_km' => $rules['vehicle']['review_mileage_km']]),
   ```

### Что уже есть на входе и чего не хватает

- ✅ `mileage` (int) — нормализован в `ApplicationValidator::validate()` и доступен в `AssessmentService` как `$input['mileage']`.
- ❌ Порог `review_mileage_km` в `config/rules.php` — нет, нужно добавить.
- ❌ Передача пробега в `DecisionEngine` — нет: текущая сигнатура `decide(float $ltv)` и конструктор с двумя LTV-порогами не предусматривают пробег.
- ❌ Учёт пробега при формировании решения — нет: вся ветка `approve` / `review` / `reject` сейчас опирается только на LTV.
