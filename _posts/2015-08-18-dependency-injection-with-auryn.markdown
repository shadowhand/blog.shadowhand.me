---
layout: post
title: Dependency Injection with Auryn
date: '2015-08-18 03:22:44'
tags:
- library
- solid
- dependency-injection
---

# Dependency Inversion is Hard

One of the hardest things to do properly when following [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles is [Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle). I struggled to understand and implement this concept for years. Often I wondered if it was really worth the trouble to write all of that extra configuration just to simplify testing. And then I stumbled on [Auryn](https://github.com/rdlowrey/Auryn) and everything changed.

## Service Locators vs Injectors

The first thing to understand about Aruyn is that it is not a service locator. This is an example of service location:

```
class UserRepository
{
    public function __construct(Container $container)
    {
        $this->container = $container;
    }

    public function getUser($id)
    {
        $db = $this->container->get('db');
        // ...
    }
}
```

The primary problem with this approach is that you have absolutely no assurances as to what will come out of the `get('db')` call. It might be slightly better if we didn't use an alias and wrote:

```
$db = $this->container->get('PDO`);
```

This still has the same problem though: there is no assurance what will come out of the locator, even if we are theoretically using a class name. A cleaner approach is to use true dependency injection. Instead of injecting a container we can inject actual class instances on construction:

```
public function __construct(PDO $db)
{
    $this->db = $db;
}
```

Now we can be absolutely sure that the database is an instance of [PDO](http://php.net/pdo). If another class is used, the type hint will cause a fatal error and we will immediately know what is wrong. When we want to test this class, simply provide a mock `PDO` object and verify that the correct calls are being made.

## Creating Instances

At this point, you may be wondering where the injected objects come from. The answer is deceptively simple: __the entire dependency graph for a class is resolved as the object is created__.

Our user repository now looks like this:

```
class UserRepository
{
    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function getUser($id)
    {
        $query = $this->db->prepare('SELECT * FROM users WHERE id = :id');
        $result = $query->execute([':id' => $id]);
        return $result->fetch();
    }
}
```

In order to connect to MySQL using PDO we need to provide a DSN and a username and password. This can be done using the `define` method:

```
$injector = new Auryn\Injector;

$injector->define('PDO', [
    ':dsn'      => 'mysql:host=localhost;dbname=demo',
    ':username' => 'demo',
    ':password' => 'secret',
]);
```

Once we have defined the PDO class, we can simply create anything that depends on it:

```
$users = $injector->make('UserRepository');
```

Notice that we didn't have to `define` the class before creating an instance of it? Auryn automatically resolves dependencies whenever it can by discovery with [Reflection](http://php.net/reflection). So long as dependencies are managed properly there is very little performance impact.

Now what happens if we have another repository such as `BookRepository`? With the current configuration, every time a `PDO` dependency is seen, a new instance will be created. To prevent this, all we have to do is tell the injector to treat it as a shared instance:

```
$injector->share('PDO');
```

Now the same instance of `PDO` will be used everywhere!

# Turtles All The Way Down

While this is the most basic usage of Auryn, even complex classes with multiple dependencies and sub-dependencies can be resolved with minimal effort. This is one of the main reasons that we choose to use Auryn when developing the [Spark](https://github.com/sparkphp/spark) framework.