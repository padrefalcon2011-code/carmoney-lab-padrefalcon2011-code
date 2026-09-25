# План: проверка «пробег ≤ 400 000 км, и��аче review» в расчёте решения

Источники правила: `docs/setup/code_map.md` (точка вставки), `AGENTS.md` (конвенции).
Проверено: `backend/config/rules.php`, `backend/src/Domain/DecisionEngine.php`,
`backend/src/Domain/AssessmentService.php`, `backend/src/Domain/ApplicationValidator.php`,
`backend/src/AppFactory.php`, `tests/Unit/DecisionEngineTest.php`,
`tests/Unit/AssessmentServiceTest.php`. Scout не привлекался —
code_map точен, все упоминания `mileage` найдены обычным grep'ом.

Семантика правила (как зафиксировано в code_map и подтверждено в коде):
пробег выше порога **всегда** понижает решение до `review`, даже если по LTV
было бы `approve`. Существующая ветка LTV-решения (`approve` / `review` /
`reject`) **не ослабляется**: при `mileage ≤ 400000` поведение по LTV
оста��тся ровно тем же.

## Файлы

По одной строке на файл. Изменения точечные, не ломают публичные контракты
сверх отдельно отмеченных сигнатур.

- `carmoney-lab/backend/config/rules.php:23` — добавить в массив `'vehicle'` ключ `'review_mileage_km' => 400000` рядом с существующим `'max_mileage_km' => 500000`. Существующий `max_mileage_km` **не трогать** — это валидационный «потолок» (500 000), отдельный от нового порога решения (400 000).
- `carmoney-lab/backend/src/Domain/DecisionEngine.php:23-28` — расширить docblock `@param` конструктора до `array{approve_max:float,review_max:float,review_mileage_km?:int}`, добавить приватное поле `private readonly int $reviewMileageKm;` и присваивание `$this->reviewMileageKm = (int) ($thresholds['review_mileage_km'] ?? 400000);` (дефолт — учебная защита, чтобы не уронить суще��твующие тесты при отсутствии ключа).
- `carmoney-lab/backend/src/Domain/DecisionEngine.php:30` — изменить сигнатуру `decide(float $ltv)` → `decide(float $ltv, int $mileage): string`; в начале метода, до проверки LTV, добавить: `if ($mileage > $this->reviewMileageKm) { return self::REVIEW; }`.
- `carmoney-lab/backend/src/Domain/AssessmentService.php:33` — обновить вызов: `$decision = $this->decisionEngine->decide($ltv, $input['mileage']);`. Поле `$input['mileage']` уже есть из `ApplicationValidator::validate()` (строка 78), дополнительных источников не требуется.
- `carmoney-lab/backend/src/AppFactory.php:37` — в DI заменить `new DecisionEngine($rules['ltv'])` на `new DecisionEngine($rules['ltv'] + ['review_mileage_km' => $rules['vehicle']['review_mileage_km']])`. Других мест сборки `DecisionEngine` в репо нет (проверено grep'ом).
- `carmoney-lab/tests/Unit/DecisionEngineTest.php:17` — обновить `setUp()` так, чтобы `review_mileage_km` попадал в пороги (например, `new DecisionEngine(['approve_max' => 65.0, 'review_max' => 85.0, 'review_mileage_km' => 400000])`), и обновить вызов `$this->engine->decide($ltv)` → `$this->engine->decide($ltv, 0)` (0 км — далеко от порога, поведение по LTV не меняется; существующие 7 кейсов в `ltvValues()` остаются корректными).
- `carmoney-lab/tests/Unit/DecisionEngineTest.php:20-38` — добавить второй `#[DataProvider]` с кейсами по новой границе 400 000 (см. раздел «Тесты»).
- `carmoney-lab/tests/Unit/AssessmentServiceTest.php:21-29` — без изменений по сигнатуре (`assess()` не меняется); но `payload()` уже передаёт `mileage: 96000`, что ниже 400 000, так что три существующих теста о��таются «зелёными». Можно дополнить новыми тестами (см. «Тесты»).

Файлы, которые **не трогаем** (прове��ено grep'ом, упоминания `mileage` есть, но трогать незачем):
- `carmoney-lab/backend/src/Domain/ApplicationValidator.php` — валидация по `max_mileage_km` (500 000) сохраняется как есть; новое правило живёт в `DecisionEngine`, а не в валидаторе.
- `carmoney-lab/backend/src/Repository/ApplicationRepository.php` — пробег сохраняется в БД как есть.
- `carmoney-lab/frontend/index.html`, `carmoney-lab/frontend/app.js` — фронт уже шлёт `mileage`, UI не меняется.
- `carmoney-lab/db/*`, `carmoney-lab/mocks/*` — пробег там не появляется.

## Шаги реализации (по порядку)

1. Прочитать текущий `carmoney-lab/backend/config/rules.php` (уже сделано), добавить строку `'review_mileage_km' => 400000,` в массив `'vehicle'` рядом с `'max_mileage_km' => 500000`. Проверить, что `<?php ... return [...]` остаётся валидным.
2. В `carmoney-lab/backend/src/Domain/DecisionEngine.php`:
   2.1. Добавить поле `private readonly int $reviewMileageKm;`.
   2.2. В конструкторе присвоить ему `(int) ($thresholds['review_mileage_km'] ?? 400000)` и расширить docblock.
   2.3. Изменить сигнатуру `decide(float $ltv)` → `decide(float $ltv, int $mileage): string`.
   2.4. В самом начале `decide()` (до ветки `if ($ltv < $this->approveMax)`) вставить проверку пробега.
3. В `carmoney-lab/backend/src/Domain/AssessmentService.php:33` обновить вызов `$this->decisionEngine->decide($ltv, $input['mileage'])`. `make lint` для `backend/`.
4. В `carmoney-lab/backend/src/AppFactory.php:37` пробросить `review_mileage_km` в `DecisionEngine`, как показано в разделе «Файлы».
5. Обновить `carmoney-lab/tests/Unit/DecisionEngineTest.php`:
   5.1. Передать `review_mileage_km` в `setUp()`.
   5.2. Заменить вызов `decide($ltv)` на `decide($ltv, 0)` в `testDecidesByLtv()`.
   5.3. Добавить `#[DataProvider('mileageValues')]` и метод `testReviewWhenMileageOverThreshold()` (см. «Тесты»).
6. Добавить в `carmoney-lab/tests/Unit/AssessmentServiceTest.php` интеграционные кейсы (см. «Тесты») — пробег именно 399 999 / 400 000 / 400 001, а также пустой `mileage` как отдельный негативный сценарий.
7. `make test` (или `vendor/bin/phpunit` в контейнере `backend`). Все 7 старых LTV-кейсов в `DecisionEngineTest` + 3 новых существующих в `AssessmentServiceTest` + новые кейсы должны быть зелёными.
8. `make lint`. Прогнать ручной `curl POST /api/applications` со снапшотом из `tests/Unit/AssessmentServiceTest::payload()` плюс `mileage=400001` — ожидаемо `decision: "review"`, `approved_limit: 0`.

## Тесты

Граничные значения (отдельными строками, как требует конвенция ��ланов):

- `mileage = 399999` при LTV=50.0 → `decision = "approve"`, `approved_limit = requested_amount`. (Под порогом — пробег не вмешивается.)
- `mileage = 400000` при LTV=50.0 → `decision = "approve"`, `approved_limit = requested_amount`. (Граница **включительно**: «не больше 400 000» → `≤`.)
- `mileage = 400001` при LTV=50.0 → `decision = "review"`, `approved_limit = 0`. (На 1 км выше — гарантированно review, даже при «зелёном» LTV.)
- `mileage` отсутствует в payload (пустой/не задан) → `ValidationException` с ошибкой по полю `mileage`, HTTP 422, решение не формируется. Это **сохраняемое** поведение сущ��ствующей валидации; в плане фиксируем явно, чтобы регрессия была видна.
- `mileage = 400001` при LTV=95.0 (выше `review_max=85.0`) → `decision = "review"` (а **не** `reject`): правило пробега срабатывает раньше LTV-ветки, как и зафиксировано в code_map.
- `mileage = 500000` (ровно старый «пото��ок» валидации) при LTV=50.0 → `decision = "approve"`: проверка пробега по-прежнему проходит (≤400000), LTV-зелёная зона.
- `mileage = 500001` → `ValidationException` 422 по `max_mileage_km` (старый потолок сохраняется), решение не формируется.

Unit-уровень (`DecisionEngineTest`):
- Новый `#[DataProvider] mileageValues` с кейсами: `399999/50.0 → approve`, `400000/50.0 → approve`, `400001/50.0 → review`, `400001/95.0 → review` (не `reject`).

Unit-уровень (`AssessmentServiceTest`):
- `testApprovesAtMileageBoundary` — `payload(amount=450000, marketValue=900000)` + `mileage=400000` → `decision=approve`, `approved_limit=450000`.
- `testReviewWhenMileageJustOverThreshold` — тот же payload, `mileage=400001` → `decision=review`, `approved_limit=0`.
- `testReviewOverridesEvenWithLowLtv` — отдельный кейс, фиксирующий, что пробег «перебивае��» LTV-зелёную зону (LTV=50, mileage=400001).
- `testEmptyMileageRejected` — payload без `mileage` → ожидать `ValidationException`; ��роверить, что в `$errors` есть ключ `mileage`.

Существующие кейсы `ltvValues` в `DecisionEngineTest` и три теста в `AssessmentServiceTest` (с `mileage=96000`, далеко ниже 400 000) **остаются валидными без изменения ожиданий** — это и есть регрессионная защита.

## Риски

- **Ломается сигнатура `DecisionEngine::decide(float $ltv)`** — новый обязательный параметр `int $mileage`. Все вызывающие места должны быть обновлены синхронно: `AssessmentService::assess()` (шаг 3) и `DecisionEngineTest::testDecidesByLtv()` (шаг 5.2). Прямых внешних потребителей `DecisionEngine` за пределами `Domain/` нет (проверено grep'ом).
- **`max_mileage_km` (500 000) и `review_mileage_km` (400 000) — два разных порога, легко пер��путать.** В плане явно разнесены: первый остаётся в `ApplicationValidator` как валидационный «потолок», второй добавляется в `DecisionEngine` как порог решения. Если кто-то «упростит» и заменит 500 000 на 400 000 в одном месте — сломается либо валидация (пробег 450 000 станет невалидным, хотя по бизне��у должен проходить и уходить в review), либо правило (пробег 450 000 даст `approve` вместо `review`).
- **Порядок проверок в `decide()` важен**: если поставить ветку пробега **после** LTV-ветки, то `mileage=400001` при LTV=95.0 уйдёт в `reject`, а не в `review` — нарушит требование «иначе review». В коде проверка пробега должна стоять **до** `if ($ltv < $this->approveMax)`.
- **Семантика границы 400 000 — «не больше» (≤)**, а не «меньше» (<). Кейс `mileage=400000` обязан остаться `approve` при зелёном LTV. В коде это `if ($mileage > $this->reviewMileageKm)`, а не `>=`.
- **Дефолт `?? 400000` в конструкторе `DecisionEngine`** — учебная страховка на случай, если ключ забудут пробросить в DI. Это маскирует ошибку конфигурации (тес��ы зелёные, в проде неожиданно). Если заказчик против дефолта — выкидываем, и тогда придётся явно с��едить, что `AppFactory` всегда передаёт ключ.
- **Изменение `decide()` ломает существующий публич��ый контракт класса.** В репозитории других вызовов нет, но если где-то в форках/инт��грациях `DecisionEngine` собирается руками — там сломается. AGENTS.md и code_map намекают, что репо учебное и форков нет, риск минимален, но отметить стоит.
- **Существующая проверка `max_mileage_km` (валидация) намеренно не меняется** — это сознательное решение, чтобы не дублировать порог и не путать «невалидный пробег» (422) с «пробег вышел в review» (200 + `decision=review`). Любой рефакторинг, объединяющий оба порога в одно поле в `rules.php`, потребует отдельного обсуждения.
- **Тесты на фронте отсутствуют** — изменение бэкенда не покрыто e2e; ручной `curl` остаётся единственной проверкой UI-стороны. Это вне нашего скоупа, но отметим.
- **Параметр `mileage` остаётся `int`** (валидатор приводит к int на строке 43 `ApplicationValidator.php`). Дробные километры не предусмотрены. Если когда-нибудь пробег станет `float` — придётся пересматривать условие.

**Что не входит в план (явно):**
- Изменение `max_mileage_km` (500 000) и текста ошибки валидации.
- Изменени�� полей в БД (`vehicles.mileage_km`) и миграций — не нужны.
- Изменение UI/фронта, текстов ошибок, REST-формата ответа.
- Перенос порога `review_mileage_km` в `rules.ltv` вместо `rules.vehicle` (можно, н�� code_map предлагает `rules.vehicle`; оставляем там).
- Реализация задачи LOAN-12 (LTV по возрас��у авто) — отдельная задача.
- Учёт пробега в `approved_limit` (сейчас лимит = 0 при `review`) — формулировка задачи про **решение**, не про лимит.
- Логирование причины `review` из-за пробега (например, в payload ответа) — текущий контракт ответа не предусматривает, добавлять не будем без запроса.
- Нагрузочные/интеграционные тесты — выходят за рамки, ограничиваемся unit + ручным `curl`.

## Вопросы заказчику (без ответа не двигаемся)

1. **Граница включит��льно или строго?** «Не больше 400 000» прочитано как `≤`. Подтвердить, что `mileage = 400000` → `approve` (а не `review`).
2. **Поведение при `mileage = 400001` и LTV > 85 (т.е. «должно быть reject»)** — оставляем `review` (правило пробега перебивает LTV) или правило применяется только к «зелёной» зоне? В плане заложен первый вариант, как и в code_map.
3. **��емантика «иначе review» при невалидном пробеге** — пробег 600 000 (выше `max_mileage_km = 500 000`) сейчас даёт 422 и никакого решения. Это сохраняем? Альтернатива — ослабить валидацию до 1 000 000 и пуска��ь в `review`. В плане — сохраняем текущее поведение, нужно подтверждение.
4. **Куда положить порог** — в `rules.vehicle` (рядом с `max_mileage_km`, как в code_map и плане) или в `rules.ltv` (рядом с порогами решения)? Выбор влияет на форму `AppFactory`-сборки и на читаемость конфига.
5. **Дефолт в конструкторе `DecisionEngine`** (`?? 400000`) — оставляем как страховку или требуем явный ключ из `rules`? В плане заложен дефолт.
6. **Нужен ли отдельный текстовый код/причина `review` в ответе** (например, `decision_reason: "mileage_over_threshold"`)? Сейчас контракт ответа не меняется.
7. **Объяснять ли правило во фронте** (подсказ��а «пробег свыше 400 000 км уходит на ручное ревью»)? В плане — не делаем, но заказчик может захотеть.
