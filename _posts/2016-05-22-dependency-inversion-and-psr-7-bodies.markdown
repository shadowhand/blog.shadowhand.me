---
layout: post
title: Dependency Inversion and PSR-7 Bodies
date: '2016-05-22 17:08:18'
tags:
- dependency-injection
- factories
- fig
- psr7
---

The last 72 hours have been interesting to say the least. [My rebuttal](http://shadowhand.me/all-about-psr-7-middleware/) to [Anthony's post](http://blog.ircmaxell.com/2016/05/all-about-middleware.html) have lead to some very interesting conversations. Among these is Andrew Carter's post entitled [PSR-7 Objects Are Not Immutable](http://andrewcarteruk.github.io/programming/2016/05/22/psr-7-is-not-immutable.html) in which he details how an exception handler middleware can generate a bad response body, if the body was already written to.

This is because the [PSR-7 StreamInterface](http://www.php-fig.org/psr/psr-7/#1-3-streams) is not immutable. As per the specification:

> Unlike the request and response interfaces, StreamInterface does not model immutability. In situations where an actual PHP stream is wrapped, immutability is impossible to enforce, as any code that interacts with the resource can potentially change its state (including cursor position, contents, and more).

The [`MessageInterface::withBody()` method](https://github.com/php-fig/http-message/blob/85d63699f0dbedb190bbd4b0d2b9dc707ea4c298/src/MessageInterface.php#L173-L186) suggests that immutability can be maintained by replacing the `StreamInterface` rather than writing to the body. Then why do we see bugs [like this one](https://github.com/zendframework/zend-expressive/issues/347)?

## The Dependency Problem

If we look at [the code in question](https://github.com/zendframework/zend-expressive/blob/29a6578fe3cc7cc0180ce146425955cab529e0b5/src/WhoopsErrorHandler.php#L83-L85) we see that Expressive writes directly to the body, rather than replacing it, effectively:

```
$response->getBody()->write($content);
```

Herein lies our dependency problem: **there is no way to create a new stream instance unless we rely on a concrete implementation**. The `StreamInterface` is mutable and has no method to create a new instance from some content. The `MessageInterface::withBody()` method requires that we pass it a stream. If we rely only on `psr/http-message` and allow the user to choose the implementation, we have no option other than to write directly to the existing stream.

Andrew states that this is a problem with PSR-7 and I don't necessarily disagree. The technical problem with streams is impossible to solve, which is why the FIG ended up with one mutable interface when all others are immutable. We cannot solve this problem directly, because that's simply the way PHP works. We can't rely on `StreamInterface::rewind()` because the stream might not be seekable.

## The Factory Solution

Anthony was entirely right when he suggested that a factory would solve the "need a clean response" problem. Andrew is highlighting something that I have wrestled with and was unable to solve. I think the correct solution is to define a factory interface for PSR-7 objects. Packages that implement PSR-7 could also implement the factory and then we could do the correct thing in our code:

```
$body = $this->factory->createStream($content);
$response = $response->withBody($body);
```

### Problems with Factories

I still stand by my earlier statement:

> The best dependency inversion we can have is not needing a container or factories.

In relation to factories, which are usually injected into the constructor, the problem is that we cannot enforce constructor arguments via an interface. Constructor signatures can be modified by extension without runtime errors. And to a lesser extent, return type enforcement is not possible before PHP 7.

While we have to acknowledge these problems, I still believe that factory is extremely useful and would be beneficial.

## Conclusion

Not only would a factory solve the partial body problem, it would provide a better focal point for continued debate on PSR-7 middleware. Having a factory would make writing middleware more robust regardless of the interface that becomes standard.