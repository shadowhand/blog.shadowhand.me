---
layout: post
title: Configuration in Spark
date: '2015-08-20 03:14:31'
tags:
- framework
- tutorial
- architecture
---

Building on my [last post about dependency injection](/dependency-injection-with-auryn) I thought it would useful to talk configuring [Auryn](https://github.com/rdlowrey/Auryn) within the context of [Spark](http://sparkphp.github.io/).

# Basic Configuration

The default configuration of Spark is in [`web/index.php`](https://github.com/sparkphp/project/blob/5f9f5eefc0a1a6120a207c4c6854b71372f312b1/web/index.php). If you were to add configuration for [PDO](http://php.net/pdo) the easiest thing to do would be:

```
$injector = new Auryn\Injector;
$injector->define('PDO', [
    ':dsn'      => 'mysql:host=localhost;dbname=demo',
    ':username' => 'demo',
    ':password' => 'secret',
]);
```

Once the injector is configured it can be passed to Spark for additional bootstrapping:

```
$app = Spark\Application::boot($injector);
```

This works well and does what we need. But we probably don't want to check this into source control with the database username and password exposed. And we might want to use a different database in development or testing than in production. A good solution to secrets is environment configuration.

## Environment Variables

There are two common packages used to for environment configuration loading: [josegonzalez/dotenv](https://github.com/josegonzalez/php-dotenv) and [vlucas/phpdotenv](https://github.com/vlucas/phpdotenv). Both are very similar, the primary difference at this time is that the package by josegonzalez is much more strict about where it declares the variables it loads from `.env`. In this example I'll be using the former.

As with any [Composer](http://getcomposer.org/) package we need to require it:

```
$ composer require josegonzalez/dotenv
```

Now we'll create a `.env` file with our database secrets in the project root:

```
db_database=demo
db_username=demo
db_password=secret
```

Since this file should never be added to git history, add it to `.gitignore` before proceeding:

```
$ echo ".env" >> .gitignore
```

Now we can use the env loader to parse this configuration in `index.php`. It is a good idea to add a check to make sure that anyone running the app will create a the `.env` file before we attempt to parse it:

```
if (!is_file($env = __DIR__ . '/../.env')) {
    throw new RuntimeException('Please create a .env file before starting the app!');
}
$env = (new josegonzalez\Dotenv\Loader($env))->parse()->toArray();
```

Now that our `.env` variables are defined in the `$env` array we can use it to configure the injector:

```
$injector->define('PDO', [
    ':dsn'      => 'mysql:host=localhost;dbname=' . $env['db_database'],
    ':username' => $env['db_username'],
    ':password' => $env['db_password'],
]);
```

This is quite simple and works okay when you only need to add one or two definitions, but what if you need to configure 5 or more classes? Or what if your test bootstrapping is different? Pretty quickly we will need a better way to configure the injector and the application. As [@elazar](https://twitter.com/elazar) pointed out to me, this can be done very cleanly with a configuration class. Let's see how it works.

## Separation of Concerns

Instead of having all our configuration directly in the front controller, let's move it into a class that can be extended for more flexibility. This class will live in `src/Configuration.php`:

```
<?php

namespace Spark\Project;

use Auryn\Injector;

class Configuration
{
    public function apply(Injector $injector, array $env)
    {
        $injector->define('PDO', [
            ':dsn'      => 'mysql:host=localhost;dbname=' . $env['db_database'],
            ':username' => $env['db_username'],
            ':password' => $env['db_password'],
        ]);
    }
}
```

Now we can replace our configuration in `index.php` by using this class instead:

```
(new Spark\Project\Configuration)->apply($injector, $env);
```

With this structure, if we had some kind of special database configuration for a testing environment we could simply create a `src/Configuration/Testing.php` file and load it separately in the test bootstrap:

```
(new Spark\Project\Configuration\Testing)->apply($injector, $testing);
```

We could even use separate `private` methods for each class and call them with `apply` for further clarity:

```
public function apply(Injector $injector, array $env)
{
    $this->configurePdo($injector, $env);
    $this->configureOauth($injector, $env);
    $this->configureFractal($injector, $env);
}

private function configurePdo(Injector $injector, array $env) { ... }
```

# Conclusion

By using environment variables and classes to apply configuration we can keep our "front matter" separated from the application. In the process we improved security by keeping our sensitive configuration out of source control and gain all the benefits of inheritance and composition for injector configuration. This same technique could be applied to routing, logging, or other dynamic aspects of application configuration.