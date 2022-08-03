# Stop abusing arrays in PHP (and start using classes)
> Originally published May 22nd 2021

My single biggest gripe with legacy PHP code is usually overuse of arrays and the pitfalls that inevitably follows from that. Arrays have their place, sure, but it seems to much like people love using them as a crutch to "just get things done".

In the following example we will attempt to improve model creation of a subset of User models, which we call a "phone user". The User model is a standard Laravel user model that you'd get from scaffolding a new app with Laravel Breeze or similar. Name is not nullable in the DB column, but we just want to make sure we can send text messages so this is only to be validated if present. We only add the "phone" string property, as a nullable field. So it is technically possible to create users with phone number set to NULL.

Now to build a solid implementation that enforces these application requirements.

### The naive array implementation
Coming out of the gate with some inputs sent to our controller, we need to either extract each property, like `$inputData['name']` as $name etc:

```php
public function createPhoneUser(Request $request)
{
    // We can't really know if this $name property is set, so default:
    $name = $request->input('name', '');

    // We gotta keep in mind that we actually pulled $name out of
    // the Request attributes, but the array key may not exist,
    // so never reference this directly.
    $user = User::create(['name' => $name]);

    return view('show.user', compact('user'));
}
```

Remember that arrays are just hash tables, keyed with integers or strings. Immediately (and repeatedly) using arrays throughout an application means you always have to keep validating and spilling details about the data structure irregardless of what you are doing.

Or we could work with the array as a data structure (still without any integrity checks):

```php
public function createPhoneUser(Request $request)
{
    $inputData = $request->all();
    // Clumsily add this, to make sure it exists:
    $inputData['name'] = $inputData['name'] ?? '';

    // At least this is safe now:
    $user = User::create($inputData);

    return view('show.user', compact('user'));
}
```

The drawback is obvious: the array does not know you want to require a string as name. And while the User model may require this (as a non nullable column), there's no good way of setting defaults without the null coalescing operator.

But we could at least decouple validation from our controller method by using a Validator:

```php
// class

private createUserRules = [
    'name' => 'string'
];

public function createPhoneUser(Request $request)
{
    // At least this is more clean:
    Validator::make($request->all(), $this->createUserRules)->validate();

    $user = User::create($request->validated());

    return view('show.user', compact('user'));
}
```

The savvy Laravelian would perhaps ask: why not use a FormRequest? In this specific example, yes that would work. But tying generic validation to requests limits us to requests, and makes this harder to write tests for. Remember that we want to reuse this without hacks.

Once validation is taken care of, how would we handle default values and mutating raw field values? Take mobile phone numbers, which can be [validated and formatted by existing libraries](https://github.com/Propaganistas/Laravel-Phone), but can come in various formats depending on if country code is present or not. A good way of dealing with mobile phone numbers from various sources would be to format these into a single format, so you don't need to handle a variable number of formats. Once persisted in the database, it can be considered validated and in a format prepended with a full country code.

### Turning the array into an object
The obvious solution is to encapsulate this in a new class and place this into the app hierarchy between Model and Controller/Repository:

```php
<?php

use Propaganistas\LaravelPhone\PhoneNumber;
use App\User;

class PhoneUser
{
    private $rules = [
        'name' => 'string',
        'phone' => 'required|phone:SE',
    ];

    private $name;
    private $phone;

    public function __construct(array $inputData)
    {
        Validator::make($inputData, $this->rules)->validate();

        $this->name = $inputData['name'] ?? '';
        $this->phone = PhoneNumber::make($inputData['phone'])->ofCountry('SE');
    }

    public function getAttributes(): array
    {
        return [
            'name' => $this->name,
            'phone' => $this->phone,
        ];
    }

    public function createPhoneUser(array $inputData)
    {
        $phoneUser = new PhoneUser($inputData);
        $user = User::create($phoneUser->getAttributes());
        return view('show.user', compact('user'));
    }
}

```

The principle advantages of this approach:

* You can encapsulate app-specific logic in PhoneUser, not tied to the Model definition itself (which should really only help the ORM handle table rows as
objects).
* It is reusable for any context, whether or not the app has a Request or running a service or maybe scheduled job.
* Using Validator facade it will still throw ValidationExceptions which can be handled by the controller method or the exception handler.
* Validating and mutating raw string values can be performed in the class.
* The raw request can be stored as an additional property and serialised for troubleshooting purposes.
* Since this is just a class, simple variations can be created by inheritance and overriding key properties. (Or if favoring a composition pattern, you could divide properties into Traits.)
