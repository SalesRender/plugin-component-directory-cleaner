# salesrender/plugin-component-directory-cleaner

Консольная команда на базе Symfony Console для рекурсивного удаления файлов и директорий старше указанного количества часов.

## Обзор

`plugin-component-directory-cleaner` предоставляет единственную консольную команду Symfony Console (`cleaner:run`), которая очищает устаревшие файлы и пустые директории по заданному пути. Команда рекурсивно обходит целевую директорию с помощью PHP-классов `RecursiveDirectoryIterator` и `RecursiveIteratorIterator`, сравнивая время модификации (`mtime`) каждого элемента с настраиваемым порогом в часах.

В экосистеме плагинов SalesRender плагины регулярно создают временные файлы -- загруженные данные, результаты экспорта, промежуточные артефакты обработки и другой эфемерный контент. Без периодической очистки эти файлы накапливаются и занимают дисковое пространство. Данный компонент решает эту проблему, предоставляя готовую к использованию консольную команду, которую можно вызывать вручную, через cron или программно.

Команда автоматически регистрируется в каждом плагине SalesRender, использующем [`plugin-core`](https://github.com/SalesRender/plugin-core), поскольку метод `ConsoleAppFactory::createBaseApp()` добавляет `DirectoryCleanerCommand` в приложение Symfony Console по умолчанию. Никакой дополнительной настройки со стороны разработчика плагина не требуется.

## Установка

```bash
composer require salesrender/plugin-component-directory-cleaner
```

> **Примечание:** Если ваш плагин зависит от [`salesrender/plugin-core`](https://github.com/SalesRender/plugin-core), данный компонент уже включён как транзитивная зависимость, и команда зарегистрирована автоматически.

## Требования

- PHP >= 7.1.0
- `symfony/console` ^5.0

## Основные классы

### `DirectoryCleanerCommand`

**Namespace:** `SalesRender\Plugin\Components\DirectoryCleaner`

**Наследуется от:** `Symfony\Component\Console\Command\Command`

Консольная команда Symfony Console, зарегистрированная под именем `cleaner:run`.

#### Конфигурация команды

| Свойство    | Значение                                               |
|-------------|--------------------------------------------------------|
| Name        | `cleaner:run`                                          |
| Description | Remove files, older than 24 hours (or another timeout) |

#### Аргументы

| Аргумент    | Режим      | Описание             | По умолчанию |
|-------------|------------|----------------------|--------------|
| `directory` | `REQUIRED` | Путь к директории    | --           |
| `hours`     | `OPTIONAL` | Таймаут в часах      | `24`         |

#### Поведение

1. Преобразует аргумент `directory` в реальный путь через `realpath()`.
2. Создаёт `RecursiveDirectoryIterator` (с флагом `SKIP_DOTS`), обёрнутый в `RecursiveIteratorIterator` с порядком обхода `CHILD_FIRST` -- это означает, что вложенные элементы обрабатываются первыми, что позволяет удалять пустые поддиректории после удаления их содержимого.
3. Итерируется по каждому элементу:
   - **Исключённые файлы** (фиксированный список: `.gitignore`) пропускаются. Вывод: `Skip [exclude]: /path/to/file`.
   - **Элементы новее порога** пропускаются. Вывод: `Skip [by timeout]: /path/to/file`.
   - **Устаревшие директории** удаляются через `rmdir()`. Вывод: `Remove [directory]: /path [Success|Failed]`.
   - **Устаревшие файлы** удаляются через `unlink()`. Вывод: `Remove [file]: /path [Success|Failed]`.
4. Возвращает код выхода `0`.

> **Важно:** Порядок обхода `CHILD_FIRST` гарантирует, что содержимое директории обрабатывается раньше самой директории. Если все файлы внутри директории устарели и удалены, `rmdir()` сможет успешно удалить опустевшую директорию. Однако если директория содержит хотя бы один неустаревший файл (или исключённый файл, например `.gitignore`), `rmdir()` завершится с ошибкой, так как директория не пуста.

## Использование

### Автономное использование

`DirectoryCleanerCommand` можно использовать в любом автономном приложении Symfony Console:

```php
<?php

use Symfony\Component\Console\Application;
use SalesRender\Plugin\Components\DirectoryCleaner\DirectoryCleanerCommand;

require __DIR__ . '/vendor/autoload.php';

$app = new Application();
// Регистрация команды очистки директорий
$app->add(new DirectoryCleanerCommand());
$app->run();
```

### С Plugin Core

При использовании [`salesrender/plugin-core`](https://github.com/SalesRender/plugin-core) (или любого из его специализированных вариантов: [`plugin-core-macros`](https://github.com/SalesRender/plugin-core-macros), [`plugin-core-logistic`](https://github.com/SalesRender/plugin-core-logistic), [`plugin-core-chat`](https://github.com/SalesRender/plugin-core-chat), [`plugin-core-pbx`](https://github.com/SalesRender/plugin-core-pbx), [`plugin-core-geocoder`](https://github.com/SalesRender/plugin-core-geocoder), [`plugin-core-integration`](https://github.com/SalesRender/plugin-core-integration)) команда `DirectoryCleanerCommand` автоматически регистрируется в `ConsoleAppFactory::createBaseApp()`:

```php
// Внутри ConsoleAppFactory::createBaseApp() из plugin-core
$app = new Application();
$app->add(new DirectoryCleanerCommand());
// ... здесь регистрируются другие команды ...
```

Типичный файл `console.php` плагина выглядит так:

```php
#!/usr/bin/env php
<?php

use SalesRender\Plugin\Core\Macros\Factories\ConsoleAppFactory;

require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/bootstrap.php';

$factory = new ConsoleAppFactory();
$application = $factory->build();
$application->run();
```

Поскольку фабрика берёт на себя регистрацию, команду `cleaner:run` можно использовать сразу, без дополнительной настройки.

### Использование из командной строки

Очистка всех файлов старше 24 часов (по умолчанию):

```bash
php console.php cleaner:run /path/to/temp/directory
```

Очистка всех файлов старше 6 часов:

```bash
php console.php cleaner:run /path/to/temp/directory 6
```

Очистка всех файлов старше 72 часов (3 дня):

```bash
php console.php cleaner:run /path/to/uploads 72
```

### Пример вывода

```
Skip [exclude]: /var/html/app/temp/.gitignore
Remove [file]: /var/html/app/temp/export_20250101.csv [Success]
Remove [file]: /var/html/app/temp/upload_20250102.xlsx [Success]
Skip [by timeout]: /var/html/app/temp/recent_file.txt
Remove [directory]: /var/html/app/temp/old_batch [Success]
```

### Планирование через cron

В плагине SalesRender можно настроить периодическую очистку с помощью метода `addCronTask()` в вашей пользовательской `ConsoleAppFactory`:

```php
use SalesRender\Plugin\Core\Factories\ConsoleAppFactory as BaseConsoleAppFactory;
use Symfony\Component\Console\Application;

class MyConsoleAppFactory extends BaseConsoleAppFactory
{
    public function build(): Application
    {
        // Запуск очистки каждый час для директории uploads
        $this->addCronTask('0 * * * *', 'cleaner:run /path/to/uploads 24');
        return parent::build();
    }
}
```

## Конфигурация

У данного компонента нет внешних конфигурационных файлов. Все параметры передаются как аргументы команды при запуске:

- **`directory`** -- абсолютный или относительный путь к очищаемой директории. Путь разрешается через `realpath()`.
- **`hours`** -- пороговое значение возраста в часах. Файлы и директории с `mtime` старше `now - hours` будут удалены. По умолчанию `24`.

Список исключений (в настоящее время только `.gitignore`) зафиксирован в методе `cleanUp()` и не может быть изменён через аргументы.

## Справочник API

### `DirectoryCleanerCommand`

| Метод | Видимость | Описание |
|---|---|---|
| `configure(): void` | `protected` | Устанавливает имя команды (`cleaner:run`), описание и аргументы (`directory`, `hours`) |
| `execute(InputInterface $input, OutputInterface $output): int` | `protected` | Точка входа; читает аргументы и делегирует выполнение методу `cleanUp()`, возвращает `0` |
| `cleanUp(string $directory, int $hoursTimeout, array $exclude, OutputInterface $output): void` | `private` | Основная логика: обходит дерево директорий в порядке child-first, удаляет устаревшие элементы, пропускает исключённые файлы |

## См. также

- [salesrender/plugin-core](https://github.com/SalesRender/plugin-core) -- Базовый фреймворк, автоматически регистрирующий эту команду через `ConsoleAppFactory`
- [salesrender/plugin-component-db](https://github.com/SalesRender/plugin-component-db) -- Связанный компонент, предоставляющий команду `db:cleaner` (`TableCleanerCommand`) для очистки старых записей в таблицах базы данных
- [Документация Symfony Console](https://symfony.com/doc/5.x/console.html) -- Справочник по используемому компоненту Symfony Console
