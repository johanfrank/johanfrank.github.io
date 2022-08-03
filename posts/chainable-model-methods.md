# Chainable model methods
> Originally published March 20th 2021

I've worked with Laravel for years now, but I still love to find ways to refine my method of working. One thing that has always bugged me is the way model properties are typically pushed around without intent being very clear.

What do I mean by intent? Just the *why* of this operation. Why am I updating a certain value - what action does that represent?

Let me demonstrate my point with a simple example. You have a Document model with a bunch of timestamps representing various events. These documents can be updated from API calls, or admin GUI actions, but whenever a API call to `POST /document/{id}/close` comes in you should set three different timestamps. And this is how you do it, in more than one place:

```php
$document = Document::findOrFail($request->get('id'));
$document->fill([
    'updated_at' => Carbon\Carbon::now(),
    'closed_at' => Carbon\Carbon::now(),
    'last_processed_at' => Carbon\Carbon::now(),
]);
$document->save();
```

Is this bad code? It's not terrible, as it runs well efficiently and fulfills the requirements of reflecting changes in this model. Long term you just have a problem of communicating intent, i.e. that
> "setting these timestamps" means "this is a closed document".

If you forget to set the last timestamp, will that break? Probably not. But it can have unforeseen consequences down the line, when you suddenly realize that 25% of all models aren't 100% correct.

So my favorite solution for this is to use something as basic as chainable model methods. How are methods made chainable? Simply by returning themselves:

```php
class Document extends Model
{
    public function close(): self
    {
        $this->updated_at = Carbon\Carbon::now();
        $this->closed_at = Carbon\Carbon::now();
        $this->last_processed_at = Carbon\Carbon::now();
        return $this;
    }
}
```

Now we can simply rewrite previous example into this:

```php
$document = Document::findOrFail($request->get('id'));
$document->close()->save();
```

The beauty of this is that it can be used for really anything regarding that model. Ideally I'd prefer to never manipulate anything model-related unless its a simple string attribute.

It also helps managing state. Lets say you have some sort of state or status column, but cleverly skipped the ENUM data type. Now how are you gonna manage whatever string you set?

```php
// Fine:
$document->status = 'closed';

// This could happen, if you don't validate using a setter:
$document->status = 'closde';
```

What if our new best friend, the chainable model method actually could handle state changes as well?

```php
class Document extends Model
{
    public function close(): self
    {
        // Only allow open or pending documents to be closed: 
        if (! in_array($this->status, ['open', 'pending'])) {
            throw new IllegalDocumentStatus('You cannot close this document.');
        }

        $this->status = 'closed';

        $this->updated_at = Carbon\Carbon::now();
        $this->closed_at = Carbon\Carbon::now();
        $this->last_processed_at = Carbon\Carbon::now();

        return $this;
    }
}
```

With this simple change, you never have to validate strings or create complex mappings of valid state changes. But for me, the biggest win is, again, intent. Instead of manually manipulating string data for this model, we can actually let the model dictate what is possible and what does actions actually mean by just good naming.
