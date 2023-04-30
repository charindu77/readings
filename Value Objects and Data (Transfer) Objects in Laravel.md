# Value Objects and Data (Transfer) Objects in Laravel
Historically, PHP has been a "loosely typed" language, with auto-converting between strings/integers and potential "magic" or bugs because of that. Slowly, the language itself evolved with type-hinting and return types, but also more people started to create their own object types, to define their object rules for minimizing bugs. These are called VALUE OBJECTS, and in this article, we'll cover when/how to use them.

## When/Why You Need a Value Object?
A quick answer would be this: you need to define the rules for the variable behavior, so if those rules are broken, PHP would throw an exception.

General string/integer/boolean variable types are often too broad to define a real-life complex business logic of the applications.

I really recommend watching the talk by Kai Sassnowski at the recent Laracon Online, where he presented three real-life examples of creating specific classes instead of integer/float/string variables.

[Video link](https://www.youtube.com/watch?v=f4QShF42c6E&t=876s)

In summary, those examples are:

- Parameter $seconds is not just an integer, it's a positive integer, so it's an object of class Duration that implements the validation of positive number;
- Parameter $discount is not just a float, it's a float between 0.01 and 1, so it's an object of class Discount that implements the necessary validation;
- Parameter $deviceID is not just a string UUID, it's a device with its unique validation rules, so it's an object of class DeviceID with the validation.
So, instead of this:
```
public function wait(int $seconds): void
{
    // ...
}
```
You should do this:
```
class Duration {
    public function __construct(public readonly int $seconds)
    {
        if ($seconds < 0) {
            throw new InvalidArgumentException();
        }
    }
 
    public static function seconds(int $seconds): self
    {
        return new self($seconds);
    }
}
 
// Then, when we use the Duration class:
public function wait(Duration $duration): void
{
    // ...
}
 
// Then, we can call it like this:
wait(Duration::seconds(5));  // will work ok
wait(-5);                    // PHP will give TYPE error
wait(Duration::seconds(-5)); // will throw our exception
```
At first glance, these may look like over-complicating things.

But if you create a function that may be used by another developer in the future (yes, including yourself), you need to make sure that parameters are passed according to the business logic rules.

If someone tries to violate those rules, intentionally or accidentally, the application should throw an exception.

In Kai's own words, "I like when tools yell at me when I'm making mistakes".

Other examples of such statements can be:

Money is not just an integer: it's an object with an amount, rounding precision, and currency
Date is not just a string: it's an object with date, format, and timezone
Person's Name is not just a string: it's an object with a first name, middle name, and last name
etc.
So, if we want to validate these values, we can of course validate them in our application code manually, with typical Laravel validation rules, or, a more strict and systemized approach would be those Value Objects classes which would contain the validation inside themselves.

The term "value object" itself is not from Laravel or PHP. You can read about its philosophy in this article by Martin Fowler

## There's No Such Thing as $data
Another reason to have more strict Value Objects is when you're dealing with a more complex structure like an unstructured array.

Have you ever seen such code?
```
public function import($data) {
    // ... importing some data from somewhere
}
```
Well, two obvious things are wrong here:

Naming of the variable $data: it should be something like $transactions or $users instead
Variable type: is it an array? A comma-separated string? Some object?
A more readable way:
```
public function import(array $transactions) {
    // ...
}
```
But even then, the array doesn't give us much information about what could be inside of that array, and what should we pass as a parameter.

Especially dealing with the import/export of CSV data, it's very easy to miss some columns, or even interchange the columns, causing all sorts of bugs.

So, similarly, like the examples above, we could introduce something like class ImportTransactionData or a single class ImportTransaction and define the structure in there.

I won't give you an example here, we'll get practical later. For now, I want you to understand the idea of why you may need Value Objects.

For more strict validation and better readability of the code in the future.

Also, Martin Joo has pointed out two more side-benefits of Value Objects, in his post Domain-Driven Design with Laravel - Value Objects:

Everything is type-hinted. No more getProduct($data) where you don't know what's gonna happen
Auto-completion. Your IDE knows that there is a startDate property of type StartDate on the DateFilter class.
Now, from "why" let's get to different options of "how".

Creating Value Objects and Laravel Custom Casting
If you have a Value Object class, your next task is to create the object from database data, right? Laravel makes it easy for you, with Custom Casts.

Generally, Casting in Laravel means auto-transforming the Eloquent field to some object type, like Carbon with Timestamp fields. Another example is Money, I have a separate long article on how to deal with Money data in Laravel.

In the official docs, you can find a separate section, specific to Value Object Casting.

Here's the example from there, showing the Address Value Object:

#### app/Casts/Address.php:
```
namespace App\Casts;
 
use App\ValueObjects\Address as AddressValueObject;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use InvalidArgumentException;
 
class Address implements CastsAttributes
{
    public function get($model, $key, $value, $attributes)
    {
        return new AddressValueObject(
            $attributes['address_line_one'],
            $attributes['address_line_two']
        );
    }
 
    public function set($model, $key, $value, $attributes)
    {
        if (! $value instanceof AddressValueObject) {
            throw new InvalidArgumentException('The given value is not an Address instance.');
        }
 
        return [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ];
    }
}
```
Then we cast to it automatically, in the User Model:
```
use App/Casts/Address;
 
class User extends Authenticatable {
 
    protected $casts = [
        'address' => Address::class
    ];
 
}
```
This example doesn't cover what's inside of the Value Object class, so let's try to construct it ourselves.

#### app/ValueObjects/Address.php:
```
class Address {
    public function __construct(public string $lineOne, public string $lineTwo)
    {
        if ($lineOne == '') {
            throw new InvalidArgumentException();
        }
 
        // ... maybe some more validation or logic
    }
}
```
A few more examples can be found in a package by Michael Rubel, called laravel-value-objects. I particularly like the practical usage of a FullName Value Object from that package:
```
$name = new FullName(' Taylor   Otwell ');
$name = FullName::make(' Taylor   Otwell ');
$name = FullName::from(' Taylor   Otwell ');
 
$name->value();   // 'Taylor Otwell'
 
$name->fullName();  // 'Taylor Otwell'
$name->firstName(); // 'Taylor'
$name->lastName();  // 'Otwell'
 
$name = 'Richard Le Poidevin';
 
$fullName = new FullName($name, limit: 2);
 
$fullName->toArray();
 
// array:3 [
//  "fullName" => "Richard Le Poidevin"
//  "firstName" => "Richard"
//  "lastName" => "Le Poidevin"
// ]
```
You can see what's actually happening inside the FullName class, here.

## What About Data Objects? Data Transfer Objects?
There are a few more similar terms like DTO (Data Transfer Objects), and also a well-known Spatie package called Laravel Data, so are those the same or not?

Yes, and no. I know, it's confusing.

You see, all those are "objects", but their structure depends on what you want to do with those objects.

Mostly, I've seen people use them for these different purposes:

- `Value Objects:` to define the value structure and ensure/validate that it's correct
- `Data Objects:` to reuse the structure, to avoid repeating it a few times in the code
- `Data Transfer Objects:` to ensure the convenient transferring of the data between sub-systems or applications, like CSV import/export
Again, the keyword here is mostly. You can find those terms online quite interchangeable and misused.

To make this matter even more confusing, Spatie themselves had a package called [data-transfer-object](https://github.com/spatie/data-transfer-object) and archived it in favor of their new package [laravel-data](https://github.com/spatie/laravel-data).

So, as the last part of this article, let's actually look at that Laravel Data package, and maybe you would be tempted to use it.

## Data Objects: Laravel Data by Spatie
If you're more of a visual learner, you can watch a lightning 15-minute talk by Freek at Laracon Online:

[lARACON VIDEO](https://www.youtube.com/watch?v=f4QShF42c6E&t=29112s)

The main benefit of the Laravel Data package is to define the Object structure ones, to be re-used in many places in your Laravel application, to avoid duplication.

Places like:

- Form Request Validation
- API Resources
- Transforming to front-end
Imagine you have this Controller method:
```
public function update(UpdateContactRequest $request, Contact $contact)
{
    $contact->update($request->validated());
 
    return back();
}
```
And this Form Request class:
```
class UpdateContactRequest extends FormRequest {
 
    public function rules() {
        return [
            'name' => ['required'],
            'email' => ['required', 'email'],
            'address' => ['required'],
            'postal' => ['required'],
            'name' => ['required'],
        ];
    }
}
```
Also, you have an API endpoint that needs to return the data with API Resource:
```
// API Controller:
public function index()
{
    return ContactResource::collection(Contact::all());
}
 
public function show(Contact $contact)
{
    return new ContactResource($contact);
}
 
// API Resource:
class ContactResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'address' => $this->address,
            'postal' => $this->postal,
            'city' => $this->city,
        ];
    }
}
```
These are different methods but they are working with the same Eloquent model fields, aren't they?

So that's where the laravel-data package comes in. You create a new PHP class, like this:

#### app/Data/ContactData.php:
```
namespace App\Data;
 
use Spatie\LaravelData\Data;
 
class ContactData extends Data
{
    public function __construct(
        public int $id,
        public string $name,
        public string $email,
        public string $address,
        public string $postal,
        public string $city,
    ) {}
 
    public function rules() {
        return [
            'name' => ['required'],
            'email' => ['required', 'email'],
            'address' => ['required'],
            'postal' => ['required'],
            'name' => ['required'],
        ];
    }
}
```
And then, you can replace both Form Request and API Resource classes, with this new ContactData class:
```
// Web Controller:
public function update(ContactData $data, Contact $contact)
{
    $contact->update($data->toArray());
 
    return back();
}
 
// API Controller:
public function index()
{
    return ContactData::collection(Contact::all());
}
 
public function show(Contact $contact)
{
    return ContactData::from($contact);
}
```
And this is just one simple example of what the laravel-data package offers, I suggest you read its full documentation.

## Conclusion: No Strings Should Be Attached
If you work with bigger projects and more complex structures in your database, you will inevitably come to the need for something like Value Objects, whether you will build some functionality yourself, or use one of the features/tools described in this article.

The main goal here is to manage the data and not let the invalid data slip through. Good luck with that!
