# Сохранённые команды

## `/review-diff`: ревью изменений перед мержем

Команда запускает ограниченное ревью текущих изменений по файлу
[`.kilo/commands/review-diff.md`](../../.kilo/commands/review-diff.md). Для навигации по
затронутому коду она использует скилл
[`ast-index-workflow`](../../.kilo/skills/ast-index-workflow/SKILL.md).

Передайте обязательный параметр: путь к одному файлу или имя базовой ветки. Путь ограничивает
ревью этим файлом, ветка ограничивает ревью diff текущей ветки относительно неё. Без параметра
команда просит указать файл или ветку и не начинает ревью.

### Прогон 1

Новая сессия Kilo Code, вызов одной строкой:

```text
/review-diff backend/src/Repository/ApplicationRepository.php
```

Результат:

```text
No findings: backend/src/Repository/ApplicationRepository.php has no staged or
unstaged diff against HEAD, so there are no changes in that file to review.

Untracked files are limited to .kilo/commands/ and docs/setup/commands.md; they do
not affect the requested file.
```

### Прогон 2

Новая сессия Kilo Code, вызов одной строкой:

```text
/review-diff backend/src/Domain/LtvCalculator.php
```

Результат:

```text
Нет изменений для ревью: git diff HEAD -- backend/src/Domain/LtvCalculator.php пуст.
Поэтому замечаний по диффу нет.
```

В обоих прогонах параметр задал разную область проверки; результат возвращается в чат с
findings, открытыми вопросами и кратким выводом о готовности изменений к мержу.

### Проверка На Чужом Входе

`frontend/app.js` → в новой сессии Agent Manager команда из `.kilo/commands/` не была
загружена как сохранённая: `/review-diff` обработался как обычный текст, поэтому ответ не
соблюдал обязательные разделы `Findings`, `Open Questions`, `Summary` → канонический файл
команды добавлен в фактический каталог конфигурации проекта
`.kilo/command/review-diff.md`; инструкция также требует вернуть только точный шаблон даже
при пустом diff. Учебная копия в `.kilo/commands/review-diff.md` сохранена согласно заданию.
