---
layout: post
title: Configuring PHP Style Checks with Composer
date: '2018-04-13 17:44:26'
tags:
- php
- style
- composer
- automation
---

**Update:** As [someone pointed out][reddit-phpcs-xml] phpcs also supports a [local configuration file][phpcs-xml] called `phpcs.xml` or `phpcs.xml.dist`. Using a local configuration file is almost certainly more efficient than using composer hooks. Use this post an example of composer hooks instead of how to configure phpcs. 🤓 This entire blog post can be replaced with the following `phpcs.xml.dist`:

```xml
<?xml version="1.0"?>
<ruleset>
    <arg name="colors"/>
    <arg value="sp"/>
    <rule ref="PSR2"/>
</ruleset>
```

[phpcs-xml]: https://github.com/squizlabs/PHP_CodeSniffer/wiki/Advanced-Usage#using-a-default-configuration-file

One of thing that has always bothered me about [phpcs][phpcs] is that the lack of a local configuration file.

[phpcs]: https://github.com/squizlabs/PHP_CodeSniffer
[reddit-phpcs-xml]: https://www.reddit.com/r/PHP/comments/8c15eg/configuring_php_style_checks_with_composer/dxb8o7f/

The official way to set the default standard for a project is:

```
vendor/bin/phpcs --config-set default_standard PSR2
```

This will write to a configuration file **inside the `vendor/`** directory, which means that the configuration cannot be committed to version control. When a new team member is added they must also run this command or different style checks will be used.

Luckily, this can be solved with [composer command events][composer-events], namely the `post-install-cmd` and `post-update-cmd` events, which can be pointed to a PHP class that processes the event. Let's create an example:

[composer-events]: https://getcomposer.org/doc/articles/scripts.md#command-events


```php
<?php
declare(strict_types=1);

namespace Acme\Extension;

use Composer\Composer;
use Composer\IO\IOInterface;
use Composer\IO\NullIO;
use Composer\Script\Event;
use Composer\Util\ProcessExecutor;

class ComposerPackageHook
{
    public static function postUpdate(Event $event): void
    {
        // add code here
    }
}
```

Now that the class has been created, let's register it in `composer.json`:

```json
{
    "scripts": {
        "post-install-cmd": [
            "Acme\\Extension\\ComposerPackageHook::postUpdate"
        ],
        "post-update-cmd": [
            "Acme\\Extension\\ComposerPackageHook::postUpdate"
        ]
    }
}
```

Note that we register both `post-install-cmd` and `post-update-cmd` to ensure that our hook will run ~~if~~ when `squizlabs/php_codesniffer` is updated, which will overwrite our configuration.

Now that the hook is defined, we can fill in the missing pieces. First, we need to check if codesniffer is installed. For that, let's add a generic helper method that can be reused later:

```php
private static function isInstalled(Composer $composer, string $package): bool
{
    $repository = $composer->getRepositoryManager()->getLocalRepository();
    $packages = $repository->findPackages($package);

    return count($packages) > 0 && $repository->hasPackage($packages[0]);
}
```

Next, we will need to execute the `config-set` commands using `phpcs`:

```php
private static function setPhpCsConfig(Composer $composer, IOInterface $io): void
{
    $vendorDir = $composer->getConfig()->get('vendor-dir');

    $io->write('Configuring phpcs for project... ', false);

    $executor = new ProcessExecutor(new NullIO());
    $executor->execute("$vendorDir/phpcs --config-set default_standard PSR2");
    $executor->execute("$vendorDir/phpcs --config-set show_progress 1");

    $io->write('done.');
}
```

Now that we have all the pieces necessary to check that codesniffer is installed (since it won't be on production) and to set the configuration, we just need to update the `postUpdate` method to make use of it:


```php
public static function postUpdate(Event $event): void
{
    $composer = $event->getComposer();

    if (self::isInstalled($composer, 'squizlabs/php_codesniffer')) {
        self::setPhpCsConfig($composer, $event->getIO());
    }
}
```

And there you have it! Now everyone on the team will be using the same style checks just by running `phpcs src/` after they install or update their local dependencies.

Here's the complete class copy/paste:

```php
<?php
/**
 * @author Woody Gilk <hello@shadowhand.me>
 * @license MIT
 * @link http://shadowhand.me/configuring-php-style-checks-with-composer/
 */
declare(strict_types=1);

namespace Acme\Extension;

use Composer\Composer;
use Composer\IO\IOInterface;
use Composer\IO\NullIO;
use Composer\Script\Event;
use Composer\Util\ProcessExecutor;

class ComposerPackageHook
{
    public static function postUpdate(Event $event): void
    {
        $composer = $event->getComposer();

        if (self::isInstalled($composer, 'squizlabs/php_codesniffer')) {
            self::setPhpCsConfig($composer, $event->getIO());
        }
    }

    private static function isInstalled(Composer $composer, string $package): bool
    {
        $repository = $composer->getRepositoryManager()->getLocalRepository();
        $packages = $repository->findPackages($package);

        return count($packages) > 0 && $repository->hasPackage($packages[0]);
    }

    private static function setPhpCsConfig(Composer $composer, IOInterface $io): void
    {
        $vendorDir = $composer->getConfig()->get('vendor-dir');

        $io->write('Configuring phpcs for project... ', false);

        $executor = new ProcessExecutor(new NullIO());
        $executor->execute("$vendorDir/phpcs --config-set default_standard PSR2");
        $executor->execute("$vendorDir/phpcs --config-set show_progress 1");

        $io->write('done.');
    }
}
```

fin!
