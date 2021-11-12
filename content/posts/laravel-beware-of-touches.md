+++
title = "Laravel: beware of $touches"
date = "2021-11-12"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Laravel", "PHP"]
keywords = ["Laravel", "PHP"]
description = "Every framework have their limits."
showFullContent = false
+++

I have been using Laravel professionally since almost 1year, and I must say: I'm very impressed with the framework.
Everything's run smoothly, there's a feature for *(almost everything)* you can think of, so you *(almost)* never need to
reinvent the wheel.

This is very advantageous since you only focus on building your product features by features and spend less time working
on technical stuff who are less business valuable.

# Everything is fine... until it's not.

---

Recently we have faced really weird MySQL error at work:

> SQLSTATE[HY000]: General error: 1390 Prepared statement contains too many placeholders

What does it mean? It is certainly obvious: *the prepared statement contains too many placeholders*.

## What are placeholders again?

Placeholder are using in SQL prepared statement as template that will be replaced by the values when the query is
executed. Example:

```sql
insert into users (username, email) values (?, ?);
```

The following query contains placeholder for username and email (identified by the '?'). When this query will be executed
the values will be replaced.

## So where's the issue?

Following the stacktrace, I've determined that the error happened when doing a `$model->save()` call. So let's analyze
the model to see if something looks off:

```php
namespace App\Models;

/**
 * @property Collection<Role> $roles
 */
class User extends Model
{
    protected $touches = ['roles'];

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}
```

As you can see the model as nothing special declared, except for a little thing: the usage of `$touches`.

### What is $touches again?

*(I apologize in advance, the example below is really poor, but I couldn't come up with something else)*

Sometime, it may be useful to bump the `updated_at` of a model: 

Let's see you are building an application to monitor the uptime of a website. Each time the website has been checked
you'll certainly want to bump the updated_at column of the Website model in order to display the value on the
interface (like a last_checked feature).

How do you touch a model to bump updated_at? Well using `$model->touch()` of course!

Okay thanks but what's with `$touches`?

![](/img/wegettherewhenwegetthere.png)

---

The role of the `$touches` variable is being able to *touch* (bump updated_at) element of a child collection when saving the
parent.

If we take our previous `User` model as example: each time you'll call `$user->save()` it will touch the roles relation (as
defined in `$touches`). Since the relation is a belongs to many it will invoke the following code:

```php
namespace Illuminate\Database\Eloquent\Relations;

class BelongsToMany extends Relation
{
    public function touch()
    {
        $key = $this->getRelated()->getKeyName();

        $columns = [
            $this->related->getUpdatedAtColumn() => $this->related->freshTimestampString(),
        ];

        // If we actually have IDs for the relation, we will run the query to update all
        // the related model's timestamps, to make sure these all reflect the changes
        // to the parent models. This will help us keep any caching synced up here.
        if (count($ids = $this->allRelatedIds()) > 0) {
            $this->getRelated()->newQueryWithoutRelationships()->whereIn($key, $ids)->update($columns);
        }
    }
}
```

This query will basically generate a single update to bump the roles.updated_at column. Something like this:

```sql
update roles set roles.updated_at = now() where roles.id in (1, 2, 3)
```

will be executed. (in this example the user has the role 1, 2 and 3 affected)

### And the problem?

Well, as you may see it coming the problem was... We have models with more than **120,000** child in their relationship.
And since Laravel is trying to execute the update request in one shot, it has encountered a MySQL limit: the placeholder
limit.

This limit in MySQL is currently at
65,535 ([see this MySQL commit](https://github.com/mysql/mysql-server/blob/3290a66c89eb1625a7058e0ef732432b6952b435/sql/sql_prepare.cc#L1505)).

## How to handle such case?

The way we have handled this situation was simply by not using `$touches`, and manually doing the touches chunk by chunk
on the roles to not reach the limit.

I have chosen to use [listeners](https://laravel.com/docs/8.x/events#registering-events-and-listeners) for that.

### Define the UserSaved event

The first thing is to create an event that will be fired when the User model is saved.

```sh
php artisan make:event UserSaved
```

and then references it in the model:

```php
namespace App\Models;

/**
 * @property Collection<Child> $childs
 */
class User extends Model
{
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
    ];

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}
```

### Define the UserListener

Then we'll need to create the listener that will handle the user events:

```php
php artisan make:listener UserListener
```

and then we'll need to listen for this particular event:

```php
namespace App\Listeners;

class UserListener
{
    public function handleUserSaved(UserSaved $event)
    {
        // Manually touches roles
        $event->user->roles()->chunk(1000, function (Collection $role) {
            Role::whereIn('id', $role->pluck('id'))->update(['updated_at' => Carbon::now()]);
        });
    }
}
```

## Isn't this workaround odd? Shouldn't $touches work out-of-the-box?

Eh, while this may be *opinionated* (but that's my blog :p). I guess yes. As a user of Laravel I would either expect:

- The framework to handle such cases
- Having a note somewhere in the docs that explain the limits of `$touches`

I have raised an [issue](https://github.com/laravel/framework/issues/39259) to laravel/framework to discuss this bug.
After a bit of discussion it has come up that fixing the framework may not be the best thing to do since this use-case
is quite rare and the fix is a bit opinionated.

Therefore, opening a [PR](https://github.com/laravel/docs/pull/7373) in laravel/docs to mention the technical limits
of `$touches`
was the logical follow-up to do. Sadly, the PR was rejected without taking time to think about it.

I must say I'm a bit disappointed of how the situation has ended, but... *meh*.

Happy hacking!