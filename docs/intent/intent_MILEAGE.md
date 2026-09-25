# Intent: проверка «пробег ≤ 400 000 км, иначе review» в расчёте решения

Источники: `docs/setup/code_map.md` (точка вставки, поведение LTV-ветки),
`docs/plan/plan_MILEAGE.md` (детальный план реализации и тестов),
`backend/config/rules.php`, `backend/src/Domain/DecisionEngine.php`,
`backend/src/Domain/AssessmentService.php`, `backend/src/Domain/ApplicationValidator.php`,
`backend/src/AppFactory.php`, `tests/Unit/DecisionEngineTest.php`,
`tests/Unit/AssessmentServiceTest.php`. Ответы заказчика в Grill Me —
2026-09-25, 5 развилок закрыты.

## Цель

В решении по заявке (`approve` / `review` / `reject`) учитывать пробег
автомобиля: при `mileage > 400 000` решение всегда понижается до `review`,
даже если по LTV было бы `approve` или `reject`. Существующая ветка
решения по LTV не ослабляется: при `mileage ≤ 400 000` поведение по LTV
остаётся ровно тем же.

## Граничные условия (закреплено заказчиком)

- Граница `400 000` — **включительно** (`≤`): `mileage = 400000` при
  зелёном LTV даёт `approve`. Реализуется условием
  `if ($mileage > $this->reviewMileageKm) { return self::REVIEW; }`.
- Проверка пробега стоит **до** ветки LTV в `DecisionEngine::decide()`.
  Поэтому `mileage = 400001` при `LTV = 95.0` (выше `review_max = 85.0`)
  даёт `review`, а **не** `reject`.
- Валидация по `max_mileage_km = 500000` сохраняется: пробег
  в `[400001, 500000]` проходит валидацию и уходит в `review`;
  пробег `500001+` → `ValidationException` → HTTP 422, решения нет.
- `mileage` отсутствует в payload → `ValidationException` 422 по полю
  `mileage` (поведение существующего `ApplicationValidator` сохраняется).

## Scope (что входит)

- `backend/config/rules.php` — добавить `'review_mileage_km' => 400000`
  в массив `'vehicle'` рядом с существующим `'max_mileage_km' => 500000`.
  Существующий `max_mileage_km` не трогаем — это валидационный «потолок»,
  отдельный от нового порога решения.
- `backend/src/Domain/DecisionEngine.php` — расширить docblock
  конструктора до `array{approve_max:float, review_max:float, review_mileage_km?:int}`,
  добавить приватное поле `private readonly int $reviewMileageKm;`,
  присвоить `(int) ($thresholds['review_mileage_km'] ?? 400000)`.
  Сигнатуру `decide(float $ltv): string` изменить на
  `decide(float $ltv, int $mileage): string`. В самом начале метода,
  до `if ($ltv < $this->approveMax)`, вставить проверку пробега.
- `backend/src/Domain/AssessmentService.php` — обновить вызов на
  `$decision = $this->decisionEngine->decide($ltv, $input['mileage']);`.
  `$input['mileage']` уже доступен из `ApplicationValidator::validate()`.
- `backend/src/AppFactory.php` — заменить
  `new DecisionEngine($rules['ltv'])` на
  `new DecisionEngine($rules['ltv'] + ['review_mileage_km' => $rules['vehicle']['review_mileage_km']])`.
- `tests/Unit/DecisionEngineTest.php` — обновить `setUp()` так, чтобы
  `review_mileage_km` попадал в пороги; в существующем `testDecidesByLtv()`
  вызов `decide($ltv)` заменить на `decide($ltv, 0)` (0 км далеко от порога,
  поведение по LTV не меняется). Добавить второй `#[DataProvider]`
  с кейсами по новой границе.
- `tests/Unit/AssessmentServiceTest.php` — добавить интеграционные
  кейсы на границе 400 000 и на пустой `mileage`. Существующие тесты
  с `mileage: 96000` остаются валидными без изменения ожиданий.

## Scope (что НЕ входит)

- Изменение `max_mileage_km` (500 000) и текста ошибки валидации.
- Изменение полей в БД (`vehicles.mileage_km`) и миграций.
- Изменение UI/фронта, текстов ошибок валидации, REST-формата ответа
  (в частности, никакого `decision_reason` в JSON).
- Учёт пробега в `approved_limit` (формулировка задачи — про решение,
  не про лимит).
- Логирование причины `review` из-за пробега.
- Реализация задачи LOAN-12 (LTV по возрасту авто).
- Перенос порога `review_mileage_km` в `rules.ltv` — оставляем в `rules.vehicle`.
- Нагрузочные и e2e-тесты — ограничиваемся unit + ручным `curl`.

## Тест-кейсы (acceptance)

Граничные значения для `DecisionEngine` (новый `DataProvider mileageValues`):

- `mileage = 399999`, LTV = 50.0 → `approve`. Под порогом — пробег
  не вмешивается.
- `mileage = 400000`, LTV = 50.0 → `approve`. Граница включительно.
- `mileage = 400001`, LTV = 50.0 → `review`. На 1 км выше —
  гарантированно `review`, даже при «зелёном» LTV.
- `mileage = 400001`, LTV = 95.0 → `review` (а **не** `reject`):
  правило пробега срабатывает раньше LTV-ветки.
- `mileage = 500000`, LTV = 50.0 → `approve`: проверка пробега
  по-прежнему проходит (≤ 400000), LTV-зелёная зона.
- `mileage = 500001` → `ValidationException` 422 по `max_mileage_km`,
  решение не формируется.

Граничные значения для `AssessmentService` (новые методы в `AssessmentServiceTest`):

- `testApprovesAtMileageBoundary` — `payload(amount=450000, marketValue=900000)`
  + `mileage=400000` → `decision=approve`, `approved_limit=450000`.
- `testReviewWhenMileageJustOverThreshold` — тот же payload,
  `mileage=400001` → `decision=review`, `approved_limit=0`.
- `testReviewOverridesEvenWithLowLtv` — LTV=50, `mileage=400001` →
  `review` (пробег перебивает LTV-зелёную зону).
- `testEmptyMileageRejected` — payload без `mileage` →
  `ValidationException`, в `$errors` есть ключ `mileage`.

Существующие 7 кейсов `ltvValues` в `DecisionEngineTest` и 3 теста в
`AssessmentServiceTest` (`mileage: 96000`) остаются валидными без
изменения ожиданий — это регрессионная защита.

## Критерии приёмки

1. `make test` (или `vendor/bin/phpunit` в контейнере `backend`) — все
   существующие и новые кейсы зелёные.
2. `make lint` (`php -l` по `backend/` и `tests/`) — без замечаний.
3. Ручной `curl POST /api/applications` со снапшотом из
   `tests/Unit/AssessmentServiceTest::payload()` плюс `mileage=400001` —
   ответ `decision: "review"`, `approved_limit: 0`, HTTP 200.
4. Тот же `curl` с `mileage=400000` — `decision: "approve"`,
   `approved_limit = requested_amount`.
5. Тот же `curl` без поля `mileage` — HTTP 422, в теле массив ошибок
   с ключом `mileage`.
6. Тот же `curl` с `mileage=500001` — HTTP 422 (поведение существующей
   валидации не сломалось).

## Ключевые риски (фиксируем явно)

- Ломается сигнатура `DecisionEngine::decide(float $ltv)` — новый
  обязательный параметр `int $mileage`. Все вызывающие места
  (`AssessmentService::assess()`, `DecisionEngineTest::testDecidesByLtv()`)
  должны быть обновлены синхронно.
- Два разных порога `max_mileage_km = 500000` (валидация) и
  `review_mileage_km = 400000` (решение) — легко перепутать.
  В коде они разнесены по разным классам (`ApplicationValidator` и
  `DecisionEngine`) и не должны «упрощаться» в одно поле.
- Порядок проверок в `decide()` важен: ветка пробега ДО LTV,
  иначе `mileage=400001` при LTV=95.0 уйдёт в `reject` вместо `review`.
- Дефолт `?? 400000` в конструкторе `DecisionEngine` — учебная страховка,
  AppFactory всё равно передаёт ключ явно.

## Файлы, которые НЕ трогаем (для ревью)

- `backend/src/Domain/ApplicationValidator.php` — валидация по
  `max_mileage_km` сохраняется как есть.
- `backend/src/Repository/ApplicationRepository.php` — пробег сохраняется
  в БД как есть.
- `frontend/index.html`, `frontend/app.js` — фронт уже шлёт `mileage`.
- `db/*`, `mocks/*` — пробег там не появляется.


Question 1: confirm inclusive boundary (≤). Plan assumes it but user asked about it.

code_map и plan_MILEAGE уже зафиксировали границу 400 000 как `≤` (mileage=400000 → approve, mileage=400001 → review) и порядок «проверка пробега ДО ветки LTV» (то есть пробег перебивает и approve, и reject в сторону review). Подтвердить как есть, или меняем?
Подтверждаю как в плане
В плане зафиксировано: пробег 500 001 (выше старого валидационного потолка max_mileage_km=500000) → ValidationException 422, решения нет; пробег не передан в payload → тоже 422 по полю mileage. Это означает, что правило «400 000 → review» срабатывает только в диапазоне [0, 500000], а 500 001+ не доходит до DecisionEngine. Подтвердить, или нужно пускать «всё, что выше 400 000» в review, ослабив валидацию?
Сохраняем 422 (Recommended)
В plan_MILEAGE явно out of scope: UI/��ронт, тексты ошибок, REST-формат ответа, БД, логирование причины review, approved_limit под пробег, нагрузочные/e2e-тесты, реализация LOAN-12. Что-то из этого реально нужно включить в эту задачу, или весь список остаётся за бортом?
Оставляем только ядро (Recommended)
В plan_MILEAGE конструктор DecisionEngine получает `review_mileage_km` с дефолтом `?? 400000` в самом классе — учебная страховка, чтобы старые тесты не падали, если ключ забыли пробросить в DI. AppFactory прокидывает ключ явно. Оставляем дефолт, или требуем строгое наличие ключа в $rules (бросаем исключение при отсутствии)?
Дефолт 400000 (Recommended)
План предлагает добавить порог в `rules.vehicle` рядом с `max_mileage_km` (логически — оба про пробег). Альтернатива — `rules.ltv` рядом с `approve_max`/`review_max` (логически — оба пороги решения). Что выбираем?
rules.vehicle (Recommended)
После подтверждения вс��х 4 пунктов картина такая: граница 400000 включительно (≤), порядок «пробег до LTV» (перебивает и reject в сторону review), валидация по max_mileage_km=500000 сохраняется (422), scope — только ядро + unit-тесты, дефолт 400000 в конструкторе, порог лежит в rules.vehicle. Этого достаточно, чтобы собрать intent в docs/intent/, или ещё что-то нужно уточнить (например, формат имени файла, дополнительные кейсы в тестах, проверка make lint/test в критериях приёмки)?
Достаточно, пиши intent