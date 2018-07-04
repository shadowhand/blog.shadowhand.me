---
layout: post
title: Immutable Data Structures in PHP
date: '2015-09-23 02:23:37'
tags:
- immutability
- data-structures
- library
---

As someone who most often works with PHP I often find myself envious of the more advanced data structures that are present in a language like Python. As an experiment, I decided to see if it would be possible to bring some of those basic structures to PHP while also preserving [immutability](https://en.wikipedia.org/wiki/Immutable_object). The result of this experiment is [Destrukt](https://github.com/shadowhand/destrukt).

## Why immutable?

Before we talk about the structures themselves, let's talk about immutability for a second. Since [PSR-7](http://www.php-fig.org/psr/psr-7/) decided to implement immutability there has a been a lot of buzz about it in the PHP community. The basic reasoning is this:

**An immutable object cannot be modified by accident.**

That might seem like a completely obvious statement or too trivial but this comes with a number of interesting "constraints" that help us write better code:

*Passing objects to collaborators is completely safe.* We no longer need to even consider the possibility that the object would be modified or put into an different state than before the call.

*Changes are explicit.* If we **do** want to allow a collaborator to modify an object, we can explicitly do so by reassignment:

```
$thing = $tustedObject->modify($thing);
```

This also has an extra benefit of making testing easier because complex objects do not need to be rebuilt as often.

*Reasoning is simple.* Because all objects are safe from modification by default, and all changes are explicit, a significant amount of [cognitive load](https://en.wikipedia.org/wiki/Cognitive_load) can be reduced by not having to follow complex method chains to see where changes occur.

Together these constraints actually give us greater freedom to do complex operations while worrying less about object state.

## What is an array?

PHP is fairly unique among popular programming languages in that the [array type](http://php.net/array) can be used in so many ways. All of the following are arrays in PHP:

```
$dictionary = [
    'cat' => 'furry pet',
    'taco' => 'delicious food',
    'bus' => 'common transportation',
];
$orderedList = ['apple', 'banana', 'banana', 'pear'];
$unorderedList = [1, 3, 5, 9, 2, 5];
$set = ['car', 'bus', 'train'];
```

While this approach provides a great amount of flexibility it also makes it more difficult to validate data that is being passed through an application. What if someone forgets that this array is meant to be associative and uses `$product[] = $description` and suddenly the array is missing the product ID as a key and gets assigned a random number that refers to some other product? Not only is this a real possibility but it is nearly impossible to discover until a customer complains that the product they are adding to their cart is changing into something else.

By using more strict data structures, we can avoid these problems and have more testable code.

## What structure?

[Destruckt](https://github.com/shadowhand/destrukt) provides the following structures:

- Dictionary, an [associative array](https://en.wikipedia.org/wiki/Associative_array) that consists of a key and a value.
- OrderedList, a [list](https://en.wikipedia.org/wiki/List_(abstract_data_type)) that contains any number of values in a defined order.
- UnorderedList, a [list](https://en.wikipedia.org/wiki/List_(abstract_data_type)) that contains any number of values in an undefined order.
- Set, a [set](https://en.wikipedia.org/wiki/Set_(abstract_data_type)) of values that contains no duplicates.

Each of these are standard data types that are commonly found in many programming languages. In destrukt, all of them can be created from a generic array and are validated. They can also be counted, serialized, encoded as JSON. Every structure is completely tested.

## How are they used?

Using the structures is extremely easy once installed. For example, this is how you create and use a set:

```
use Shadowhand\Destruckt\Set;

$set = new Set(['apple', 'banana', 'pear']);
```

Now we have a full featured set object that can checked for values:

```
$set->hasValue('apple'); // true
$set->hasValue('orange'); // false
```

And if we want to add a new value, we create a copy with the new value:

```
$copy = $set->withValue('orange');
$copy->hasValue('orange'); // true
$set->hasValue('orange'); // false
```

Or if we want to overwrite the object with the copy, we just reassign it:

```
$set = $set->withValue('orange');
```

Once the set is created, we can just as easily remove values:

```
$set = $set->withoutValue('orange');
```

If we add a duplicate value to the set, nothing will be changed:

```
$copy = $set->withValue('apple');
var_dump($copy === $set); // true
```

But what if we try to create a set with duplicate values?

```
$set = new Set(['car', 'car', 'truck']);
```

~~This will throw an `InvalidArgumentException` because a set can only contain unique values! This ensures that the structures are always in a consistent state and helps prevent mistakes in application logic.~~

As of version `0.3.0`, Set automatically removes duplicate values on create or when using `withData`. As was [pointed out to me on r/php](https://www.reddit.com/r/PHP/comments/3m0uea/immutable_data_structures_in_php/cvbvujp) this is more consistent with other languages that have native implementations of sets.

### Where's the data?

When you want to extract the underlying data from the structure, there are multiple options:

```
$data = $set->getData(); // array
$data = $set->toArray(); // alias for getData
$json = json_encode($set);
$frozen = serialize($set);
```

### And the others?

The other data structures have very similar usage, with some minor differences. For instance, the `Dictionary` type takes a key and a value:

```
$dict = (new Dictionary)->withValue('dog', 'fetches ball');
```

The usage of the structures should be fairly intuitive and you can always read [the tests](https://github.com/shadowhand/destrukt/tree/master/tests) for complete usage examples.

## Conclusion

More advanced data structures offer some nice benefits and more safety when compared to simple arrays. By using immutability, our code is easier to reason about and less prone to side effects. Together immutable data structures offer significant benefits over using plain arrays to pass data through your application.