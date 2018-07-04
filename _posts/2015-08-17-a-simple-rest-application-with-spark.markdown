---
layout: post
title: A simple REST application with Spark
date: '2015-08-17 04:34:46'
tags:
- framework
- tutorial
- rest
---

# What is Spark?
[Spark](http://sparkphp.github.io/) is a [PSR-7](http://www.php-fig.org/psr/psr-7/) micro-framework that implements the [Action Domain Responder](http://pmjones.io/adr/) pattern created by [@pmjones](https://twitter.com/pmjones). ADR can be summarized as a "clean code" version of MVC, with specific differences that help us achieve [SOLID](http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles.

## Install Spark

This assumes that you already have [Composer](https://getcomposer.org/) installed.

```
$ composer create-project sparkphp/project demo
```

This will create a new Spark project in the `demo` directory.

## Application Layout

Spark has a very simple filesystem structure:

- `src/` where your application code goes
- `web/` public web directory, including the [front controller](http://www.martinfowler.com/eaaCatalog/frontController.html)
- `vendor/` dependencies installed with composer

This layout provides a very clear separation between public and server code and follows standard conventions.

## Routing

Since Spark is a web framework, it requires routing. By default, routes are defined in `web/index.php` and look something like:

```
$app->addRoutes(function(Spark\Router $r) {
    $r->get('/hello[/{name}]', 'Spark\Project\Domain\Hello');
});
```

The Spark application is represented by `$app` and the `addRoutes` method takes a [closure](http://php.net/closure) that receives the `Router` when the application is ready to begin routing. The router is based on [FastRoute](https://github.com/nikic/FastRoute) with some convenience added by having separate methods for creating different HTTP requests.

The first parameter of `get` is the URL path that will be matched. Anything wrapped with `[brackets]` is optional and anything wrapped with `{braces}` is a variable parameter.

The second parameter of `get` is the name of the Domain class that will be used to handle the request.

## Try It Out

First fire up the built in PHP web server:

```
$ php -S localhost:8000 web/index.php
```

Now you should be able to visit <http://localhost:8000/hello> and see a simple "hello, world" message as a JSON response by using the class in `src/Domain/Hello.php`. When a name is added, such as <http://localhost:8000/hello/sally>, it will be used instead of "world".

## Add another domain

Add a new route in `web/index.php`:

```
$r->get('/friends[/{name}]', 'Spark\Project\Domain\Friends');
```

And create the domain class in `src/Domain/Friends.php`:

```
<?php
namespace Spark\Project\Domain;

use Spark\Adr\DomainInterface;
use Spark\Payload;

class Friends implements DomainInterface
{
    public function __invoke(array $input)
    {
        $friends = [];
        $mood    = 'sad';
        $name    = 'me';

        if (!empty($input['friends'])) {
            $friends = $input['friends'];
            $mood    = 'happy';
        }

        if (!empty($input['name'])) {
            $name = $input['name'];
        }

        return (new Payload)
            ->withStatus(Payload::OK)
            ->withOutput([
                'who'     => $name,
                'mood'    => $mood,
                'friends' => $friends,
            ]);
    }
}
```

Spark merges all query parameters, post body, cookies, and attributes such as route parameters into `$input` before passing it to the domain. This allows us to pass the `friends` through the query string: <http://localhost:8000/friends/jimmy?friends[]=amanda>.

# Conclusion

This is only a starting point but hopefully it gives a sense at how easy REST applications are to create with Spark.