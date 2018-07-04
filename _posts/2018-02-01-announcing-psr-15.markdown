---
layout: post
title: Announcing PSR-15
featured: true
date: '2018-02-01 21:25:00'
tags:
- php
- fig
- middleware
- psr7
- psr15
---

As of January 22nd, 2017 [PSR-15: HTTP Server Request Handlers][psr-15] has been accepted. The [final vote][acceptance-vote] was 12 members in favor, none opposed, and none abstaining. I am very grateful to everyone that contributed during the last (almost) two years. And especially thankful for [Matthew Weier O'Phinney][mwop], who ultimately sponsored the PSR and got it through the review phase!

The PSR itself is split into two parts: a middleware interface and a request handler interface. Typically the middleware will perform checks or modifications on the request, pass it to the request handler to generate a response, and then perform checks or modifications on the response before returning it.

The [psr/http-server-middleware][http-server-middleware] package provides the following interface:

```php
namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;

interface MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface;
}
```

The [psr/http-server-handler][http-server-handler] provides this interface:

```php
namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

interface RequestHandlerInterface
{
    public function handle(
        ServerRequestInterface $request
    ): ResponseInterface;
}
```

Writing a middleware is done by importing the `MiddlewareInterface` and defining a `process()` method. This method must generate a response, either by delegating to the request handler or generating it internally. For example, if a middleware was checking [CORS][cors-wiki] headers it would generate a response internally if the request did not have correct headers. If the correct headers are present, it would delegate to the request handler to generate a response. [Matthew Weier O'Phinney][mwop] has a much more [in-depth example of middleware][mwop-psr-15] on his blog, along with a lot of historical information.

Writing a request handler is less common. Typically the request handler will be implemented in a middleware dispatching system to continue processing a middleware stack. The request handler may also be used as the last item in a middleware stack to execute application code. It is not expected that controllers or domain actions will implement the request handler, though it is possible to do so.

A final note: as of the time of writing this post, just 10 days after the acceptance vote, there are already 107 packages depending on [psr/http-server-middleware][http-server-middleware] and over 2000 installs. This is a great vote of confidence from the community!

[acceptance-vote]: https://groups.google.com/d/msg/php-fig/f5lL_QNIrgI/SmYZVw_5AwAJ
[cors-wiki]: https://en.wikipedia.org/wiki/Cross-origin_resource_sharing
[http-server-handler]: https://packagist.org/packages/psr/http-server-handler
[http-server-middleware]: https://packagist.org/packages/psr/http-server-middleware
[mwop]: https://twitter.com/mwop
[mwop-psr-15]: https://mwop.net/blog/2018-01-23-psr-15.html
[psr-15]: https://www.php-fig.org/psr/psr-15/
