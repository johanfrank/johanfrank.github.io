# Disabling submit button on browser validation failures
> Originally published February 22nd 2022
> 
Disabling your submit button in your HTML form, so your users can't accidentally post the same contents more than once, is quite simple using Alpine.js.
Lets just bind a `@click` to a simple boolean model and have that toggle:

```html
<form name="myForm" x-data="{ disabled: false }">
    <input name="first_name" type="text" required>
    <input type="submit" x-bind:disabled="disabled" @click="disabled = !disabled">
</form>
```

Really simple... until you forget to fill out that required form! But oh the state is messy. And you don't want to skip the browser validation right... 😩

Luckily there's a [Constraint Validation API](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation#the_constraint_validation_api) available in modern browsers! And as long as your form is named you can access this through `document.forms`, and hey presto there's even a method that returns `true` if all your form elements are valid:

```html
<form name="myForm" x-data="{ disabled: false }">
    <input name="first_name" type="text" required>
    <input type="submit" x-bind:disabled="disabled" @click="disabled = document.forms.myForm.checkVal
</form>
```
