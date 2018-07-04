---
layout: post
title: Using Auryn to Write Factories
date: '2015-12-16 19:24:06'
tags:
- dependency-injection
- architecture
- service-location
- factories
- routing
---

A while back I write [Dependency Injection with Auryn](http://shadowhand.me/dependency-injection-with-auryn/) which covered high level configuration of classes. It did not properly discuss how to handle situations where service location is necessary.

## But Service Locators Are Evil!

You'll often hear this statement when talking with someone about dependency injection. In fact, Auryn even has this bold warning in the [README](https://github.com/rdlowrey/auryn/blob/master/README.md#how-it-works):

> auryn **is NOT** a Service Locator

This is the pattern that is maligned and results in untestable code:

```
function __construct(Container $container)
{
    $this->db = $container->get('database');
}
```

You should never, ever write this kind of code. If you discover it in a project, refactor it out as soon as possible.

## A Necessary Evil

Sometimes service location is necessary to ensure that only the code needed is being loaded and executed. Routing to a controller based on the URL path is an obvious real world situation. We might have some routing code that looks like this:

```
$router->get('/home', 'HomeController');
$router->get('/login', 'LoginController');
$router->post('/login', 'LoginController');
```

In this scenario, we don't want to load `LoginController` when requesting the home page. The only way to handle this is with service location.

### Doing It Right

How can we accomplish this properly with Auryn? The answer is to use a [`callable` factory](http://php.net/manual/en/language.types.callable.php) that allows us to write a **specific** factory for controllers. It might look something like this:

```
$injector = new Auryn\Injector;
$router->factory(function ($controller) use ($injector) {
    // If the $controller was a fully qualified class name, this step could be skipped
    $class = 'Acme\\Controller\\' . $controller;
    return $injector->make($class);
});
```

Now the router could have some code that makes use of the factory internally:

```
public function route($url)
{
    foreach ($this->routes as $route) {
        if ($route->matches($url)) {
            return $this->execute($route->controller(), $route->method(), $this->factory);
        }
    }
}

private function execute($class, $method, callable $factory)
{
    $controller = $factory($class);
    return call_user_func([$controller, $method]);
}
```

Success! We've written service location without leaking access to the injector and it is fully testable if we mock the factory.

## Conclusion

[Auryn](https://github.com/rdlowrey/Auryn) is not a service locator but it makes it really easy write proper, testable service locators.