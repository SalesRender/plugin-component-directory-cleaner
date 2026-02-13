# salesrender/plugin-component-directory-cleaner

Symfony Console command that recursively deletes files and directories older than a specified number of hours.

## Overview

`plugin-component-directory-cleaner` provides a single Symfony Console command (`cleaner:run`) that cleans up stale files and empty directories from a given path. It traverses the target directory recursively using PHP's `RecursiveDirectoryIterator` and `RecursiveIteratorIterator`, comparing each item's modification time (`mtime`) against a configurable timeout threshold in hours.

In the SalesRender plugin ecosystem, plugins frequently generate temporary files -- uploaded data, export results, intermediate processing artifacts, and similar ephemeral content. Without periodic cleanup, these files accumulate and consume disk space. This component solves that problem by providing a ready-to-use console command that can be invoked manually, via cron, or programmatically.

The command is automatically registered in every SalesRender plugin that uses [`plugin-core`](https://github.com/SalesRender/plugin-core), since `ConsoleAppFactory::createBaseApp()` adds `DirectoryCleanerCommand` to the Symfony Console application by default. No additional configuration is required on the plugin developer's side.

## Installation

```bash
composer require salesrender/plugin-component-directory-cleaner
```

> **Note:** If your plugin depends on [`salesrender/plugin-core`](https://github.com/SalesRender/plugin-core), this component is already included as a transitive dependency and the command is auto-registered.

## Requirements

- PHP >= 7.1.0
- `symfony/console` ^5.0

## Key Classes

### `DirectoryCleanerCommand`

**Namespace:** `SalesRender\Plugin\Components\DirectoryCleaner`

**Extends:** `Symfony\Component\Console\Command\Command`

A Symfony Console command registered under the name `cleaner:run`.

#### Command Configuration

| Property    | Value                                                  |
|-------------|--------------------------------------------------------|
| Name        | `cleaner:run`                                          |
| Description | Remove files, older than 24 hours (or another timeout) |

#### Arguments

| Argument    | Mode       | Description          | Default |
|-------------|------------|----------------------|---------|
| `directory` | `REQUIRED` | Path to directory    | --      |
| `hours`     | `OPTIONAL` | Timeout in hours     | `24`    |

#### Behavior

1. Resolves the `directory` argument to a real path via `realpath()`.
2. Creates a `RecursiveDirectoryIterator` (with `SKIP_DOTS` flag) wrapped in a `RecursiveIteratorIterator` using `CHILD_FIRST` order -- meaning deepest items are processed first, allowing empty subdirectories to be removed after their contents.
3. Iterates over every item:
   - **Excluded files** (hardcoded list: `.gitignore`) are skipped. Output: `Skip [exclude]: /path/to/file`.
   - **Items newer than the threshold** are skipped. Output: `Skip [by timeout]: /path/to/file`.
   - **Stale directories** are removed with `rmdir()`. Output: `Remove [directory]: /path [Success|Failed]`.
   - **Stale files** are removed with `unlink()`. Output: `Remove [file]: /path [Success|Failed]`.
4. Returns exit code `0`.

> **Important:** The `CHILD_FIRST` iteration order ensures that a directory's contents are processed before the directory itself. If all files inside a directory are stale and removed, `rmdir()` can then successfully remove the now-empty directory. However, if the directory contains even one non-stale file (or an excluded file like `.gitignore`), `rmdir()` will fail because the directory is not empty.

## Usage

### Standalone Usage

You can use `DirectoryCleanerCommand` in any standalone Symfony Console application:

```php
<?php

use Symfony\Component\Console\Application;
use SalesRender\Plugin\Components\DirectoryCleaner\DirectoryCleanerCommand;

require __DIR__ . '/vendor/autoload.php';

$app = new Application();
// Register the directory cleaner command
$app->add(new DirectoryCleanerCommand());
$app->run();
```

### With Plugin Core

When using [`salesrender/plugin-core`](https://github.com/SalesRender/plugin-core) (or any of its specialized variants like [`plugin-core-macros`](https://github.com/SalesRender/plugin-core-macros), [`plugin-core-logistic`](https://github.com/SalesRender/plugin-core-logistic), [`plugin-core-chat`](https://github.com/SalesRender/plugin-core-chat), [`plugin-core-pbx`](https://github.com/SalesRender/plugin-core-pbx), [`plugin-core-geocoder`](https://github.com/SalesRender/plugin-core-geocoder), [`plugin-core-integration`](https://github.com/SalesRender/plugin-core-integration)), the `DirectoryCleanerCommand` is automatically registered in `ConsoleAppFactory::createBaseApp()`:

```php
// Inside plugin-core ConsoleAppFactory::createBaseApp()
$app = new Application();
$app->add(new DirectoryCleanerCommand());
// ... other commands registered here ...
```

A typical plugin's `console.php` looks like this:

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

Since the factory handles registration, you can immediately use `cleaner:run` without any additional setup.

### CLI Usage

Clean all files older than 24 hours (default):

```bash
php console.php cleaner:run /path/to/temp/directory
```

Clean all files older than 6 hours:

```bash
php console.php cleaner:run /path/to/temp/directory 6
```

Clean all files older than 72 hours (3 days):

```bash
php console.php cleaner:run /path/to/uploads 72
```

### Example Output

```
Skip [exclude]: /var/html/app/temp/.gitignore
Remove [file]: /var/html/app/temp/export_20250101.csv [Success]
Remove [file]: /var/html/app/temp/upload_20250102.xlsx [Success]
Skip [by timeout]: /var/html/app/temp/recent_file.txt
Remove [directory]: /var/html/app/temp/old_batch [Success]
```

### Scheduling with Cron

In a SalesRender plugin, you can schedule periodic cleanup using `addCronTask()` in your custom `ConsoleAppFactory`:

```php
use SalesRender\Plugin\Core\Factories\ConsoleAppFactory as BaseConsoleAppFactory;
use Symfony\Component\Console\Application;

class MyConsoleAppFactory extends BaseConsoleAppFactory
{
    public function build(): Application
    {
        // Run cleanup every hour for the uploads directory
        $this->addCronTask('0 * * * *', 'cleaner:run /path/to/uploads 24');
        return parent::build();
    }
}
```

## Configuration

This component has no external configuration files. All parameters are passed as command arguments at runtime:

- **`directory`** -- the absolute or relative path to the directory to clean. The path is resolved via `realpath()`.
- **`hours`** -- the age threshold in hours. Files and directories with `mtime` older than `now - hours` will be deleted. Defaults to `24`.

The exclusion list (currently only `.gitignore`) is hardcoded in the `cleanUp()` method and cannot be changed via arguments.

## API Reference

### `DirectoryCleanerCommand`

| Method | Visibility | Description |
|---|---|---|
| `configure(): void` | `protected` | Sets command name (`cleaner:run`), description, and arguments (`directory`, `hours`) |
| `execute(InputInterface $input, OutputInterface $output): int` | `protected` | Entry point; reads arguments and delegates to `cleanUp()`, returns `0` |
| `cleanUp(string $directory, int $hoursTimeout, array $exclude, OutputInterface $output): void` | `private` | Core logic: iterates the directory tree in child-first order, removes stale items, skips excluded files |

## See Also

- [salesrender/plugin-core](https://github.com/SalesRender/plugin-core) -- Core framework that auto-registers this command via `ConsoleAppFactory`
- [salesrender/plugin-component-db](https://github.com/SalesRender/plugin-component-db) -- Related component providing the `db:cleaner` command (`TableCleanerCommand`) for cleaning old database table records
- [Symfony Console Documentation](https://symfony.com/doc/5.x/console.html) -- Reference for the underlying Symfony Console component
