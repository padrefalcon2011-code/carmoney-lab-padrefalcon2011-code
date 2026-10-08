# Замеры токенов: 2.1.1--2.1.2

Промпт трека А: «Какие пороги и правила про пробег реально зашиты в коде и где они лежат?»

| Замер | input | output | $ | Агент звал ast-index | Команды ast-index |
|---|---:|---:|---:|---|---|
| Без индекса | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | требуется подтвердить по ленте сессии | — |
| ast-index | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | да | `ast-index stats`; `ast-index search "mileage"`; `ast-index search "max_mileage_km"` |
| ast-index + Caveman | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | требуется значение из OpenRouter Activity | да | `ast-index stats`; `ast-index search "mileage"`; `ast-index search "max_mileage_km"` |

## Проверка способа поиска

После установки агент вызвал `ast-index` и получил точные места для `mileage` и
`max_mileage_km`. Затем прочитал только найденные файлы `backend/config/rules.php`
и `backend/src/Domain/ApplicationValidator.php`, а не весь репозиторий.

Числа input, output и $ нужно перенести из OpenRouter → Activity для двух новых
сессий с MiniMax M3. Их нельзя достоверно получить из локального лога или
восстановить после сессии.

## Проверка сжатого ответа: 2.1.2

Ответ Caveman на тот же вопрос сохранил все факты из ответа с индексом:
`backend/config/rules.php:23` и `max_mileage_km => 500000`, проверку диапазона
`0 <= mileage <= max_mileage_km` и текст ошибки в
`backend/src/Domain/ApplicationValidator.php:44-45`, отсутствие действующих
правил LTV по пробегу с TODO LOAN-12 в
`backend/src/Domain/AssessmentService.php:12`, хранение `mileage_km` в
`db/schema.sql:22`, маппинг в
`backend/src/Repository/ApplicationRepository.php:38-45` и оба тестовых
значения: `84000` в `tests/Unit/ApplicationValidatorTest.php:34`, `96000` в
`tests/Unit/AssessmentServiceTest.php:38`.

Вывод: по содержанию ничего не потерялось: все релевантные файлы, проверки и
значения на месте. Экономию в input/output/$ можно определить только после
внесения показателей Activity; без них нельзя честно назвать число сэкономленных
токенов или решить, что дало больше -- индекс или Caveman.

## Итоговый замер: 2.1.4

Для сценария с пробегом `400001` агент выбрал `ast-index` и Playwright. Для
вопроса о пороге пробега агент вызвал только `ast-index`, без Playwright.

Сравнение со строкой «Без индекса» из таблицы выше выполнить численно нельзя:
в обеих строках `input`, `output` и `$` отмечены как требующие значений из
OpenRouter Activity. Локальная лента сессий не содержит этих показателей, поэтому
разницу токенов и стоимость не подставляем.
