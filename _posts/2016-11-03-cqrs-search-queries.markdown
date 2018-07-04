---
layout: post
title: 'CQRS: Search Queries'
date: '2016-11-03 20:25:09'
tags:
- php
- architecture
- cqrs
- easydb
- dto
---

[CQRS](http://martinfowler.com/bliki/CQRS.html) has been getting a fair bit of attention in the last few months. One of the major benefits of using CQRS is that you can completely isolate reading data from writing data. This allows architecting around [slices instead of layers](https://vimeo.com/131633177). The less cross-cutting concerns exist in your application the more rapidly change can be realized.

# Creating a Query Object

Having an object that represents a query is a great starting point for experimenting with CQRS. Let's we have a database of users in our system and an administrator can search for users based on a set of criteria:

- The first name of the user
- The last name of the user
- The role of the user

In addition, we allow the results to be sorted by first or last name.

_**Note:** These examples are not going to address limits or offsets. There are multiple ways to handle paging and addressing them would outside the scope of this post._

From this information, we can construct `UserSearchQuery` that represents the criteria:

```
final class UserSearchQuery
{
    public static function forOwner(
        $first_name = null,
        $last_name = null,
        $role = null,
        $order_by = null
    ) {
        return new self(
            $first_name,
            $last_name,
            $role,
            $order_by ?: 'last_name'
        );
    }

    private $first_name;
    private $last_name;
    private $role;
    private $order_by;

    public function firstName()
    {
        return $this->first_name;
    }

    public function lastName()
    {
        return $this->last_name;
    }

    public function role()
    {
        return $this->role;
    }

    public function orderBy()
    {
        return $this->order_by;
    }

    private function __construct(
        $first_name,
        $last_name,
        $role,
        $order_by
    ) {
        $this->first_name = $first_name;
        $this->last_name = $last_name;
        $this->role = $role;
        $this->order_by = $order_by;
    }
}
```

There are a couple of things to note right away. The first is that we are using a [named constructor](http://verraes.net/2014/06/named-constructors-in-php/) and making the `__construct` function private. This allows us to expose different variations of the search criteria in different contexts without having to refactor code using the class. For instance, we might want to have a `forEmployee` method that eliminates the ability to search by `$role`:

```
$query = SearchUserQuery::forEmployee($first_name, $last_name);
```

The second thing to note is that this class is completely immutable. There are no methods that allow it to be modified, the `__construct` method is private, and the class is marked as `final`. This makes it a safe [DTO](http://martinfowler.com/eaaCatalog/dataTransferObject.html), which is exactly what we want, since a query will be carrying data into your [domain layer](https://en.wikipedia.org/w/index.php?title=Domain_layer).

By itself this is a good start but we can improve it even more by adding state validation. One way to accomplish this is with the [beberlei/assert](https://github.com/beberlei/assert) package. Let's update our constructor with validation:

```
use function Assert\lazy as asserts;

final class UserSearchQuery
{
    // ...

    private function guardState()
    {
        asserts()

            ->that($this->first_name, 'first_name')
                ->nullOr()->string()

            ->that($this->last_name, 'last_name')
                ->nullOr()->string()

            ->that($this->role, 'role')
                ->nullOr()->inArray(['employee', 'manager', 'owner'])

            ->that($this->order_by, 'order_by')
                ->nullOr()->inArray(['first_name', 'last_name'])

            ->verifyNow();
    }
}
```

If anything goes wrong in this check, a `Assert\LazyAssertionException` will be thrown that contains all the validation errors. We can now call this function from the constructor to ensure that our object cannot be created with invalid state:

```
final class UserSearchQuery
{
    // ...

    private function __construct(
        $first_name,
        $last_name,
        $role,
        $order_by
    ) {
        $this->first_name = $first_name;
        $this->last_name = $last_name;
        $this->role = $role;
        $this->order_by = $order_by;

        $this->guardState();
    }

    // ...
}
```

# Creating a Repository Object

By itself, a query object does nothing but capture and validate some user request parameters. In order to respond to the request we will need to query the database. A fantastic PDO wrapper called [EasyDB](https://github.com/paragonie/easydb) will make this very straight forward.

```
use ParagonIE\EasyDB\EasyDB;
use ParagonIE\EasyDB\EasyStatement;

class SearchUserRepository
{
    private $db;

    public function __construct(EasyDB $db)
    {
        $this->db = $db;
    }

    public function search(SearchUserQuery $query)
    {
        $where = EasyStatement::open();

        if ($query->firstName() || $query->lastName()) {
            $whereName = $where->group();

            if ($query->firstName()) {
                $whereName->with('first_name LIKE ?', '%' . $query->firstName() . '%');
            }

            if ($query->lastName()) {
                $whereName->orWith('last_name LIKE ?', '%' . $query->lastName() . '%');
            }
        }

        if ($query->role()) {
            $where->with('role = ?', $query->role());
        }

        $query = "SELECT * FROM users WHERE $where ORDER BY {$query->orderBy()}";

        return $this->db->run($query, $where->values());
    }
}
```

Perfect, now we have a repository that takes the query object and returns an array of database results. All of the query parameters are escaped safely thanks to EasyDB and creating the dynamic conditions for first/last name was simple with EasyStatement. Even the direct usage of the `$query->orderBy()` is safe, because we know the query object validation limits the possibilities to `first_name` or `last_name`, which are valid column names.

The query generated would be something like the following, depending on the parameters set in the query object:

```
SELECT *
FROM users
WHERE (
  first_name LIKE '%luke%'
  OR last_name LIKE '%sky%'
)
AND role = 'employee'
ORDER BY last_name
```

_Note: we will assume that this class will be dependency injected with a working `EasyDB` connection._

## Escaping LIKE Conditions
**Update:** There is [one additional consideration](https://www.reddit.com/r/PHP/comments/5b18jc/cqrs_search_queries/d9ldrlu/) that we need to address here, which is that `LIKE` conditions have additional characters that must be escaped. This can be done with the `escapeLikeValue()` method in EasyDB. The first and last name query conditions become:

```
if ($query->firstName()) {
    $whereName->with('first_name LIKE ?', '%' . $db->escapeLikeValue($query->firstName()) . '%');
}

if ($query->lastName()) {
    $whereName->orWith('last_name LIKE ?', '%' . $db->escapeLikeValue($query->lastName()) . '%');
}
```

# Using the Query Object

Now that we have a query object we can use it from our controller or whatever the incoming request is processed:

```
$query = SearchUserQuery::forOwner(
    $this->request->get('first_name'),
    $this->request->get('last_name'),
    $this->request->get('role'),
    $this->request->get('order_by')
);
```

And once we have the query we can pass it into a repository to get the results. For the sake of simplicity, assume that `$this->users` is an instance of `SearchUserRepository` that was dependency injected into the controller:

```
$users = $this->users->search($query);

// process/format the results however you need to
```

# Conclusion

With this example we have seen that creating query objects can be simple, without the need for complex modeling systems. Using a self-validating object creates a safe contract between our application and our domain. Using a simple (but powerful!) PDO wrapper makes writing dynamic queries simple, without the need for an ORM package.

If you have thoughts about this article, be sure to tweet me [@shadowhand](https://twitter.com/shadowhand)!