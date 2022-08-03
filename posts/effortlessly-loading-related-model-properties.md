# Effortlessly loading related model properties
> Originally published April 18th 2021

How often do you do this? From my current project, the Organisation model has a User relationship, so to output a related model property you always need to do 1+1 query:

```php
{{ $organisation->owner->name; }}
```

This generates these queries:

```sql
select * from `organisations` where `organisations`.`id` = 1 limit 1;
select * from `users` where `users`.`id` = 1 limit 1;
```

### "Why don't you just JOIN the users table?"
Doing joins on tables are pretty simple, but the result is a bit muddled. At first we are firmly in Eloquent land, working with ORM models, but this quickly becomes very close to pure SQL:

```php
$organisation = Organisation::leftJoin('users', 'organisations.user_id', '=', 'users.id')
    ->addSelect(['*', 'users.name AS user_name'])
    ->find($id);
```

But at least now we can change our reference and keep the query count down:

```php
{{ $organisation->user_name }}
```

A few issues have crept into this solution however:

1. Refactoring user and organisation relationships suddenly require manual updates of this construct.
2. Since Eloquent basically translates Query objects to SQL, and then runs this SQL query, this means that sometimes introducing new columns with the same name can cause unresolvable queries that suddenly fail ("property is ambiguous").

### Adding a property using subselect
Now, with Eloquent being an SQL ORM, we will need to compromise: maybe we can avoid having basically "inline SQL" but still reference Eloquent model properties?

Instead of joining the table, maybe we can use subqueries to fetch data? Since the subquery can reference the "outer query", we can actually use the magic of Eloquent to join on keys:

```php
$organisation = Organisation::addSelect('*')
    ->selectSub(
        User::select('name')->whereColumn('id', 'user_id')->getQuery(),
        'user_name' // this becomes our alias
    )
    ->find($id);
```

The corresponding SQL helps explain how simple this is:

```sql
SELECT
    *,
    (
        SELECT
            `name`
        FROM
            `users`
        WHERE
            `id` = `user_id`
    ) AS `user_name`
FROM
    `organisations`
WHERE
    `organisations`.`id` = 1
LIMIT
    1
```

You can then quite simply extract this into a powerful local query scope:

```php
class Organisation
{
    public function scopeWithUserProperty(Builder $query, string $userProp, string $asAttribute)
    {
        return $query
            ->addSelect('*')
            ->selectSub(
                User::select($userProp)->whereColumn('id', 'user_id')->getQuery(),
                $asAttribute
            );
    }
}
```

And the usage gets quite expressive:

```php
$organisation = Organisation::withUserProperty('name', 'user_name')->find($id);
```

### Caveats
Before using any of these solutions, you should consider the following:

1. We have "polluted" the model with an undocumented property (user_name). Pushing this Organisation model around could cause issues. Using this consistently may require you to extend your projects Model base class to list "virtual properties", so these in fact become documented and their intent clarified.
2. Using the selectSub() solution requires careful usage of select() when building queries. By default, the Builder object does SELECT * FROM tbl and once you started manipulating this in your builder you need to make sure you are not overriding, instead extending this list of columns.
