---
layout: post
title: Announcing Equip Framework
date: '2016-05-31 13:19:00'
tags:
- php
- library
- framework
- equip
---

Last summer the development team at [When I Work](http://wheniwork.com/) started to develop an open source framework that would be the basis for future projects and provide a migration path from our aging [Kohana](http://kohanaframework.org/) codebase to something more modern.

Originally we called it [Spark](https://github.com/sparkphp). While we liked the idea of "starting something" it quickly became apparent that due to the [plethora](https://duckduckgo.com/?q=spark+programming) of projects already using the name Spark, not to mention a [long standing Apache project](https://spark.apache.org/), it was a poor choice.

The name [Equip](http://equip.github.io/) was chosen instead because it also represented something we value: being able to pick and choose useful tools without getting stuff we don't need.

If frameworks were knives, Equip would be a [Santoku](https://en.wikipedia.org/wiki/Santoku) and most other frameworks would be a [Swiss Army Knife](https://en.wikipedia.org/wiki/Swiss_Army_knife). Without doubt, both will cut things, but the Santoku will be really effective at what it does. While multi-purpose tools are perfectly acceptable when you are prototyping and aren't sure what your business requirements will be in a week or a month or a year, it makes sense to choose something that offers a lot of flexibility and defines your architecture for you.

## Standing on the Shoulders of Giants

Equip may be new but it is inventing very little. Like any good modern PHP framework it borrows from the best of the ecosystem in creative ways. The basic design of Equip is based around the [ADR pattern](http://pmjones.io/adr/) with [Auryn](https://github.com/rdlowrey/Auryn) making [dependency injection much easier](http://shadowhand.me/dependency-injection-with-auryn/). We use [FastRoute](https://github.com/nikic/fastroute) for routing, [Relay](http://relayphp.com/) for middleware dispatching, and [Whoops](https://filp.github.io/whoops/) for exception handling.

Optional support for [DotEnv](https://github.com/josegonzalez/php-dotenv), [Monolog](https://github.com/Seldaek/monolog), [Plates](http://platesphp.com/), and [Redis](http://redis.io/) is included and can be enabled when you need it.

Since Equip is based around [PSR-7](http://www.php-fig.org/psr/psr-7/) we require an implementation that includes server requests. You can choose which vendor you want to use. Our current preference is [Diactoros](https://github.com/zendframework/zend-diactoros) but Equip should work equally well with [Slim](http://www.slimframework.com/) if you wanted to mix frameworks.

## The Bootstrap

I am really proud of our bootstrap. While some frameworks have big bootstraps and multitudes of configuration files, Equip has just [`public/index.php`](https://github.com/equip/project/blob/11261d41cf89789ca1cd500bb6685a7dfea52040/public/index.php). It looks like this:

```php
<?php

// Include Composer autoloader
require __DIR__ . '/../vendor/autoload.php';

// Set the root namespace for your project
use Equip\Project\Domain;

Equip\Application::build()
->setConfiguration([
    Equip\Configuration\AurynConfiguration::class,
    Equip\Configuration\DiactorosConfiguration::class,
    Equip\Configuration\EnvConfiguration::class,
    Equip\Configuration\MonologConfiguration::class,
    Equip\Configuration\PayloadConfiguration::class,
    // Equip\Configuration\PlatesConfiguration::class,
    // Equip\Configuration\PlatesResponderConfiguration::class,
    // Equip\Configuration\RedisConfiguration::class,
    Equip\Configuration\RelayConfiguration::class,
    Equip\Configuration\WhoopsConfiguration::class,
])
->setMiddleware([
    Relay\Middleware\ResponseSender::class,
    Equip\Handler\ExceptionHandler::class,
    Equip\Handler\DispatchHandler::class,
    Equip\Handler\JsonContentHandler::class,
    Equip\Handler\FormContentHandler::class,
    Equip\Handler\ActionHandler::class,
])
->setRouting(function (Equip\Directory $directory) {
    return $directory
        ->get('/', Domain\Welcome::class)
        ->get('/hello[/{name}]', Domain\Hello::class)
        ->post('/hello[/{name}]', Domain\Hello::class)
        ; // End of routing
})
->run();
```

As a developer, you have complete control over this file. You can add new configuration, change middleware, and modify routing all in one. Configuration is just a list of classes, as is middleware. Both are executed in the order defined. Routing is done with a closure or any callable. Each of these steps are collected and then executed when `run()` is called.

### Configuration

The first step of Equip bootstrapping is configuration of Auryn for dependency injection. Each of these classes receive the injector instance and apply specific configuration. If there are variable settings that are based on deployment, such as database credentials or service tokens, support for `.env` files is provided by the `Equip\Env` class. Injecting the `Env` class is super simple with the `Equip\Configuration\EnvTrait`.

### Middleware

The next step of bootstrapping is to collect middleware. The middleware list is executed in the order added. Typically the last middleware is `ActionHandler`, which executes the domain code that was selected by routing and passes the payload generated by the domain to one or more formatters that prepare a PSR-7 response from the payload. Finally, the `ResponseSender` translates the response object into an HTTP response by sending the headers and body content.

### Routing

The final step of bootstrapping is routing. The `Directory` is passed to a closure, which defines the actions that are responsible for specific HTTP method and URI combinations. Typically the action is defined just as the domain class that should be executed. It it also possible to pass a fully constructed `Action` class for control over the input that the domain receives and the responder for domain output.

### Domains

In Equip, all application logic is executed with `Domain` classes. The domain receives all of the request input as a single `$input` array and returns a `Payload` object. The payload can contain a status, messages, and output that will ultimately be translated into a HTTP response. Within domain code there are no HTTP references. Authentication is handled in middleware, as is parsing request bodies. This allows domain code to be reused in other contexts and greatly simplifies testing.

### Execution

When the `run()` method is called, configuration is applied to the injector and the middleware is executed by Relay. All of the heavy lifting is handled by dependency injection and middleware. Very simple and, in our experience, extremely effective.

### Read the Docs

For more in depth explanation of how bootstrapping and the rest of Equip works, be sure to [read the docs](https://equipframework.readthedocs.io/en/latest/)!

## Present and Future

Late last week, we released [Equip version 2.0.0](https://packagist.org/packages/equip/framework). This is a significant milestone for us, as it represents nearly a year of refinement, enhancement, and tuning. Fundamentally, very little has changed since version 1.0.0. Our core dependencies haven't changed and domain level interfaces remain exactly the same.

I am quite proud of what we have accomplished since the last major release and feel that Equip is now a mature and stable micro-framework.

Looking forward, there are no set plans to make additional changes. At this point, it is unlikely that the framework will radically change until the PHP FIG [standardizes a HTTP middleware interface](http://shadowhand.me/all-about-psr-7-middleware/). Even then, this will be mostly changes in configuration. Domain classes written today will likely work for years to come.

## Equip More Tools

Equip provides a number of additional packages that can be useful in high performance applications, such as:

- [a data layer](https://github.com/equip/data)
- [commands](https://github.com/equip/command)
- [redis queues](https://github.com/equip/redis-queue)
- [session management](https://github.com/equip/session)
- and [authentication](https://github.com/equip/auth).

As When I Work continues to build production applications with Equip more tools may appear. We're currently working on some interesting REST authentication solutions that may be partially open sourced.

## Conclusion

As your application grows so will your team. The struggle to provide high performance applications always comes up against architecture problems. Equip provides a minimal, effective, and clean structure without trying to dictate how your business logic should work. We hope it can be as powerful for you as it has for us.