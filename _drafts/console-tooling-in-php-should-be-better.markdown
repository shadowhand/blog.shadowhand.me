---
layout: post
title: Console Tooling in PHP Should Be Better
---

Like many people who use PHP, I have a need to write console tools from time to time. The de facto standard is [symfony/console](https://symfony.com/doc/current/console.html). While there [are a number of packages matching "console"](https://packagist.org/search/?q=console), none come close to offering the features, flexibility, and power of `symfony/console`. This is not a post about how to use `symfony/console` though. This is a post about why I wish there was an alternative.

The single biggest annoyance I have with symfony/console is:

**It does not provide good [Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control).**

The typical way to write a console application looks something like this:

```
$app = new Application('User Tool');
$app->addCommands([
    new LookupCommand(),
    new BanCommand(),
    new UnbanCommand(),
]);
$app->run();
```

What arguments are available for `LookupCommand`? Who knows, all the configuration is in the command, so we need to execute it or open the class definition to find out. What happens when you execute `run()`? Who knows, all the argument parsing and exception handling is done inside the application.

These commands will probably need to read information from the database. How does the database instance get injected? If we want to do [Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) properly we need to *work around* Console, rather than with it.

If we are practicing [DDD](https://en.wikipedia.org/wiki/Domain-driven_design) then these commands are probably duplications of existing functionality in our ~~web~~ primary application. It would be far better if we could execute existing code rather than wrapping it up in `Command` classes.

I really want to be able to do something like this:

```
$app = new Application('User Tool');
$app->setOutput(new BufferedOutput());

$app->addCommand('lookup')
    ->addParameter('username', Parameter::REQUIRED)
    ->setHandler(Lookup::class);
// ...

try {
    $target = $app->parse($argv);
    $handler = $container->get($target->getHandler());
    $params = $target->getParams();
    $ = $handler->execute($params->getParameter('username'));
} catch (Exception $e) {
    $app->writeException($e);
    exit(1);
}

$app->write("user {$user->
```
