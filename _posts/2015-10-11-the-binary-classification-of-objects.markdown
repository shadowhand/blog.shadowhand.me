---
layout: post
title: The Binary Classification of Objects
date: '2015-10-11 17:09:46'
tags:
- data-structures
- architecture
- patterns
---

After working on [immutable data structures](immutable-data-structures-in-php) and [dependency injection with Auryn](dependency-injection-with-auryn) I often find myself questioning class constructor style when doing code reviews. Recently I was reviewing [a pull request](https://github.com/sparkphp/auth/pull/2) that had this signature in it:

```
public function __construct(
    RequestInterface $request,
    DateTime $now,
    $key,
    $ttl,
    $algorithm = 'HS256'
) { ... }
```

## A Funny Smell

Something immediately struck me as wrong about this signature and it took me a while to figure out what it was:

**Mixing classes and anything else in a constructor is weird.**

This reflects a core concept of object oriented programming:

> Objects sometimes correspond to things found in the real world.

The simple things can be described without further reduction:

```
class Circle
{
    public function __construct(
        $radius
    ) { ... }
}
```

This is a classic example of a value object. Another might be:

```
class Token
{
    public function __construct(
        $value,
        $expiration
    ) { ... }
}
```

Now what happens if we have a higher level class that is harder to define in simple terms?

## The Power of Value Objects

Sharing configuration between different classes is a great way to improve code. For instance, a token generator:

```
class TokenGenerator
{
    public function __construct(
        $timestamp,
        $key,
        $ttl = 60,
        $algorithm = 'HS256'
    ) { ... }

    public function getToken(User $user) { ... }
}
```

In this situation, how valuable is it that we test every permutation of the constructor arguments? How would we test the difference between a `ttl=60` and `ttl=120`? In the case of JWT we can decode the output and verify exact matches, but how valuable would that be? In my opinion, it is far more valuable to:

1. Defines what configuration is necessary to generate and validate tokens.
2. Ensure that token generation uses configuration correctly.

Starting from this position it is clear that the configuration would be better defined as a [value object](https://en.wikipedia.org/wiki/Value_object) than as raw values. This changes our signature to:

```
class TokenGenerator
{
    public function __construct(
        TokenConfiguration $config
    ) { ... }

    public function getToken(User $user) { ... }
```

Along with the new configuration object:

```
class TokenConfiguration
{
    public function __construct(
        $timestamp,
        $key,
        $ttl,
        $algorithm
    ) { ... }
}
```

Now we have a clearly defined value object that can be reused for both generation **and** validation without having to duplicate the values. If we need to add more options to the configuration we can do so without worrying that extended classes will no longer match signatures. And testing the generator becomes much easier because we can provide a [mock object](https://en.wikipedia.org/wiki/Mock_object) of the configuration to assert that the correct values are used at the correct time.

## Binary Class Pattern

What I conclude from working through this particular code review and my own projects is that we can write better code by choosing to use only two types of objects:

1. Value Objects, constructed only with scalar values.
2. Collaborator Objects, constructed only with other objects.

# Conclusion

By following a binary class pattern our code becomes easier to understand and more adaptable to change while also being more robust. We gain the ability to do independent testing of operation and structure while maintaining a clearer separation of concerns.