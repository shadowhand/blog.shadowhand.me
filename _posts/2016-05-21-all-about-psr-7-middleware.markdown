---
layout: post
title: All About PSR-7 Middleware
date: '2016-05-21 14:53:41'
tags:
- dependency-injection
- fig
- middleware
- http
- psr7
---

On May 10th, 2016, [I proposed an HTTP Middleware](https://github.com/php-fig/fig-standards/pull/755) PSR to [PHP FIG](http://www.php-fig.org/). Since then, there has been a significant amount of concern around one aspect of the proposal in particular, which [Anthony Ferrara](http://www.ircmaxell.com/) has summarized quite well in his blog post [All About Middleware](http://blog.ircmaxell.com/2016/05/all-about-middleware.html).

Before we get too deep into why Anthony and I disagree, take a look at the proposed interface as it exists right now:

```
<?php

namespace Psr\Http\Middleware;

use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

interface MiddlewareInterface
{
    /**
     * Process a request and return a response.
     *
     * Takes the incoming request and optionally modifies it before delegating
     * to the next handler to get a response. May modify the response before
     * ultimately returning it.
     *
     * @param RequestInterface $request
     * @param ResponseInterface $response
     * @param callable $next delegate function that will dispatch the next middleware component:
     *  function (RequestInterface $request, ResponseInterface $response): ResponseInterface
     *
     * @return ResponseInterface
     */
    public function __invoke(
        RequestInterface $request,
        ResponseInterface $response,
        callable $next
    );
}
```

If you have worked with a PSR-7 compatible microframework, you have almost certainly seen this signature already:

```
fn(request, response, next): response
```

## What's to fight about?

Without question, the most divisive part of the proposed interface has the inclusion of the `$response` parameter. The perceived problem, as Anthony puts it, is this:

> What does `$response` mean inside of the middleware?

From my perspective, the answer to this question is quite simple: The response is a instance of a response that may be clean or may have been modified.

Anthony does not disagree on this point and goes on to say:

> The problem is that the actual meaning of the instance passed in **depends on what outer middleware (middleware that was called before it) decided the meaning should be**. This means that no middleware can actually trust what `$response` means.

And there is the source of the disagreement. I believe that we can, and *should* trust that the middleware we *choose* to use is doing the right thing. If we are unsure, we can easily validate it ourselves.

## The Middleware Lifecycle

At this point, I think it would be useful to explain how the middleware lifecycle works, as I and others choose to implement it. Here is a very succinct diagram that comes from [Slim Framework](http://www.slimframework.com/docs/concepts/middleware.html): 

![Middleware Diagram](//www.slimframework.com/docs/images/middleware.png)

In terms of code, the basic premise is that you create a request instance, a response instance, and pass them into some kind of dispatcher that holds you middleware. In [Equip](https://equipframework.readthedocs.io/en/latest/) the middleware is added via an application builder, in the `index.php` front controller:

```
<?php

Equip\Application::build()
// ...
->setMiddleware([
    Relay\Middleware\ResponseSender::class,
    Equip\Handler\ExceptionHandler::class,
    Equip\Handler\DispatchHandler::class,
    // ...
    Equip\Handler\ActionHandler::class,
])
// ...
->run();
```

The middleware is executed in the order added, so the `ResponseSender` is both the first entry and final exit point. The `ActionHandler` is executed last and returns first. All of these middleware components **only depend on PSR-7 interfaces, _not_ a specific implementation of those interfaces**.

## The World of HttpFoundation

Those that are opposed to the current proposal are almost universally users of Symfony `HttpFoundation` or their exposure to middleware comes from [StackPHP](http://stackphp.com/) which is based on Symfony `HttpKernel`. These users have, in my estimation, become used to the idea that they can create custom extensions of a concrete class and use them everywhere. If your entire object-oriented HTTP world is based around knowing that you can create an extension of `Symfony\Component\HttpFoundation\Response` and everything just works, then losing control of that ability seems scary. 

The world we have with PSR-7 is better, precisely because we no longer need to rely on extension and concrete implementations. Instead, by inverting the choice of implementation to the developer using these components, we allow for greater freedom of choice and better interoperability. If the proposed PSR becomes a standard, anyone will be able to write middleware that type hints against a standard interface. Anyone who wants to use that middleware will be able to do so, with an entirely different implementation of PSR-7 than the author. Imagine a world where there is no "Slim middleware" or "Guzzle middleware", only "HTTP middleware". Anyone who uses PSR-7 will be able to choose from a huge ecosystem, similar to our huge ecosystem of Composer packages. We won't need to have 50 different implementations of HTTP caching that do the same thing, we can have a handful that serve specific needs.

But what about these pesky partial responses that are going to trip us up?

## Exceptions in the Lifecycle

Time and time again, the concern brought up about not being able to trust the state of the response is raised. Some have described a situation where an error occurs somewhere during the middleware lifecycle and a partially complete or error response is generated with incorrect headers attached. Anthony uses the cache middleware as an example of this, because it applies cached headers to the response before executing the rest of the middleware stack. What Anthony and others do not consider is that there would almost certainly be an `ExceptionHandler` middleware that wraps later middleware.

Having an `ExceptionHandler` as part of your middleware stack is quite common. A typical exception handler middleware will look something like this:

```
<?php

class ExceptionHandler
{
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response,
        callable $next
    ) {
        try {
            return $next($request, $response);
        } catch (Exception $e) {
            return $this->withException($response, $e);
        }
    }

    private function withException(ResponseInterface $response, Exception $e)
    {
        $response = $response->withStatus(500);
        // ... additional code to write response body
        return $response;
    }
```

The critical thing to note here is that **the partial response is never used**. If an exception occurs, the response that was passed to this middleware is decorated and returned. So long as your middleware stack has this middleware as close to the top as possible, a badly formatted response will never get decorated as an error. We don't need a factory to achieve this, because the functionality is already there. We don't need a concrete implementation, because it would hurt interoperability. **The best dependency inversion we can have is not needing a container or factories.**

## Conclusion

And there you have it. These concerns about trust really boil down to not trusting that a developer knows what they are doing. By taking that stance we are throwing out the baby with the bath water. By inverting more control to developers, while providing an interface that reinforces this inversion, we are increasing the ability to share. This is good for interoperability and good for the PHP ecosystem. The front page of PHP FIG states the mission as:

> Moving PHP forward through collaboration and standards.

This middleware standard would do both by building on the foundation of PSR-7 to enable further collaboration in the realm of middleware.
