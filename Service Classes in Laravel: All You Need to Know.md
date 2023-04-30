# Service Classes in Laravel: All You Need to Know
Service classes a very popular in Laravel projects. In this tutorial, I will explain what is a Service, when/how to use it, and what should NOT be done in Services.

## What is a Service Class?
A service class is just a standalone PHP class that holds the business logic. It typically doesn't extend any other class. Usually, it is named with a suffix of Service for example UserService.

Quite often, Service class is created for additional extra logic related to a specific Eloquent model, like UserService for a User model. Other cases are about specific functionality "topic" like PaymentService.

Below are three typical examples of Services and methods.
```
namespace App\Services;
 
class UserService {
 
    public function store(array $userData): User
    {
        // Code to Create User and return User.
    }
}
```
```
namespace App\Services;
 
class CartService
{
    public function getFromCookie()
    {
        // Get Cart and return it.
    }
}
```
```
namespace App\Services;
 
class PaymentService
{
    public function charge($amount)
    {
        // Charge User
    }
}
```
## Why Services and Not Models?
If the Service is related to Eloquent Model, then it's fair to say that you could put all that logic in the Model itself.

It's more of a personal preference, but if we are talking about bigger projects, the Model class can become a really big file with 1000+ lines.

In my philosophy, I consider a Model as kind of a "settings class" on top of Eloquent. So it should contain everything related to fields, database tables, relationships, and some attributes with accessors and mutators.

Even the words: compare the word Model to the word Service. If you try to imagine what is Model in real life, it's something static, like a sculpture, and Service means some action.

For example, we have an Order Model which has an Invoice. If we put that extra logic in the Model itself...

*app/Models/Order.php:*
```
class Order extends Model
{
    protected $fillable = ['user_id', 'details', 'status'];
 
    public function invoice(): HasOne
    {
        return $this->hasOne(Invoice::class);
    }
 
    public function pushStatus(int $status): void
    {
        $this->update(['status' => $status]);
 
        // Maybe some other actions?
    }
 
    public function createInvoice(): Invoice
    {
        if ($this->invoice()->exists()) {
            throw new \Exception('Order already has an invoice');
        }
 
        return DB::transaction(function() {
            $invoice = $this->invoice()->create();
            $this->pushStatus(2);
 
            return $invoice;
        });
    }
}
```
In the controller it could be something like the below:

*app/Http/Controllers/Api/InvoiceController.php:*
```
class InvoiceController extends Controller
{
    public function store(Order $order)
    {
        try {
            $invoice = $order->createInvoice();
        } catch (\Exception $exception) {
            return response()->json(['error' => $exception->getMessage()], 422);
        }
 
        return $invoice->invoice_number;
    }
}
```
In this case, it's not much code, but creating an invoice could have more logic, like sending notifications, creating shipment information, etc. Then it means the Model file would grow and grow.

So, it would be better to add this logic into a service:

*app/Services/OrderService.php:*
```
class OrderService
{
    public function pushStatus(Order $order, int $status): void
    {
        $order->update(['status' => $status]);
 
        // Maybe some other actions?
    }
 
    public function createInvoice(Order $order): Invoice
    {
        if ($order->invoice()->exists()) {
            throw new \Exception('Order already has an invoice');
        }
 
        return DB::transaction(function() use ($order) {
            $invoice = $order->invoice()->create();
            $this->pushStatus($order, 2);
 
            return $invoice;
        });
    }
}
```
And then in the controller:

*app/Http/Controllers/Api/InvoiceController.php:*
```
class InvoiceController extends Controller
{
    public function store(Order $order)
    {
        try {
            $invoice = $orderService->createInvoice($order);
        } catch (\Exception $exception) {
            return response()->json(['error' => $exception->getMessage()], 422);
        }
 
        return $invoice->invoice_number;
    }
}
```
## Why not Repositories?
Historically, Repository pattern has been used in Laravel to create the methods of manipulating data on top of Eloquent Model, with the idea that Some Repository could be replaced easily with Another Repository in the future.

But it makes much more sense when you use programming languages or frameworks that don't have an Eloquent ORM mechanism.

In the case of Laravel, Eloquent itself is the layer between the Controller and Database, so it kinda acts as Repository Pattern already.

Instead of
```
SELECT * FROM USERS;
```
you write
```
User::all();
```
That's why adding a Repository as another layer on top of already a repository-like layer doesn't make much sense, in my opinion, and doesn't give many benefits.

I have a course How to Structure Laravel Projects which has a lesson called Repositories: Why NOT to Use Them?.

And I have a YouTube video on this topic called [Laravel Code Review: Why NOT Use Repository Pattern?](https://www.youtube.com/watch?v=giJcdfW2wC8)

## How to Create a Service Class
There is no php artisan make:service command, you have to do it manually.

Usually, services are put into the app/Services directory, so first you would need to create the Services directory inside app and then manually create your service PHP class.

Here's an example screenshot from the IDE.

![image](https://user-images.githubusercontent.com/11309713/235370840-9b4eb090-d542-4a5c-9cdc-7c91be3f1730.png)

## How to Call Service Classes from Controllers
I will show you two ways.

We can use Service classes with or without dependency injection. Let's say we have a UserService with a store method.
```
namespace App\Services;
 
class UserService {
 
    public function store(array $userData): User
    {
        // Code to Create User and return User.
    }
}
```
In the controller, we have a store() method which is called after pressing submit button in the form.

First, let's look at how we would use it without dependency injection. We just create an object class wherever we need it, using new keyword and call the store() method.
```
class UserController extends Controller
{
    public function store(UserStoreRequest $request)
    {
        (new UserService())->store($request->validated());
 
        return redirect()->route('users.index');
    }
}
```
If you have more actions in that Service to be called, you can use a variable:
```
class UserController extends Controller
{
    public function store(UserStoreRequest $request)
    {
        $userService = new UserService();
        $userService->store($request->validated());
        $userService->sendGreetingsEmail($request->email);
        // ... $userService->whateverElse();
 
        return redirect()->route('users.index');
    }
}
```
Now, with dependency injection, Laravel can auto-create our Service object in Controllers, it's also called "auto-resolving".

In the store() method, we pass UserService as an argument, providing its type. The controller would look like this:
```
class UserController extends Controller
{
    public function store(UserStoreRequest $request, UserService $userService)
    {
        $userService->store($request->validated());
 
        return redirect()->route('users.index');
    }
}
```
In my opinion, it's clearer because the body of the method remains shorter and more readable, without initializations of the classes.

## Tip: Services Should Not Work with Global Values
Generally, Service class and its methods are kind of like a "black box":

- you input some parameters from controller
- service methods do some job
- and return the result to controller
In other words, Service shouldn't "know" about any global things like Auth, Session, Request URL, etc. Also, it shouldn't return the response to the browser.

Also, Controller is called a Controller for a reason, it needs to control the output. For example, the below code in service should throw an exception instead of aborting.
```
class VoteService
{
    public function store($question_id, $value): Vote
    {
        $question = Question::find($question_id);
        abort_if(
            $question->user_id == auth()->id(),
            500,
            'The user is not allowed to vote to your question'
        );
        // ...
    }
}
```
And then that exception may be caught in the controller or in the general exception handler of Laravel. So it could look like this:
```
class VoteService
{
    public function store($question_id, $value): Vote
    {
        $question = Question::find($question_id);
        if ($question->user_id == auth()->id()) {
            throw new \Exception('The user is not allowed to vote to your question');
        }
        // ...
    }
}
```
And then we catch the exception in the controller:
```
class VoteController extends Controller
{
    public function voice(StoreVoteController $request, VoteService $service)
    {
        try {
            $voice = $service->store($request->input('question_id'), $request->input('value'));
        } catch (\Exception $e) {
            abort(500, $ex->getMessage());
        }
        // ...
    }
}
```
So the controller should control the flow and the service should just throw an exception.

Another non-ideal thing in this example: Service uses the auth() global helper. But as I said earlier, Service is a black box and it shouldn't know anything the whole Laravel project, global variables, users, session, etc. So here we should pass ID as a parameter.
```
class VoteService
{
    public function store($question_id, $value, $user_id): Vote
    {
        $question = Question::find($question_id);
        if ($question->user_id == $user_id) {
            throw new \Exception('The user is not allowed to vote to your question');
        }
        // ...
    }
}
```
```
class VoteController extends Controller
{
    public function voice(StoreVoteController $request, VoteService $service)
    {
        try {
            $voice = $service->store($request->input('question_id'), $request->input('value'), auth()->id());
        } catch (\Exception $e) {
            abort(500, $ex->getMessage());
        }
        // ...
    }
}
```
## Tip: Services should be Reusable
Services can be called not from Controller necessarily. It could be called in an Artisan command or a Unit Test.

Let's say we can a CurrencyService which converts currency.
```
class CurrencyService
{
    const RATES = [
        'usd' => [
            'eur' => 0.98
        ]
    ];
 
    public function convert(float $amount, string $currencyFrom, string $currencyTo): float
    {
        $rate = self::RATES[$currencyFrom][$currencyTo] ?? 0;
 
        return round($amount * $rate, 2);
    }
}
```
It can be easily unit tested, without touching the DB or simulating the browser/API request. We can just call the method, pass some input and check the output result.
```
class CurrencyTest extends TestCase
{
    public function test_convert_usd_to_eur(): void
    {
        $priceUSD = 100;
 
        $this->assertEquals(98, (new CurrencyService())->convert($priceUSD, 'usd', 'eur'));
    }
 
    public function test_convert_gbp_to_eur(): void
    {
        $priceUSD = 100;
 
        $this->assertEquals(0, (new CurrencyService())->convert($priceUSD, 'gbp', 'eur'));
    }
}
```
I also have a [YouTube video](https://www.youtube.com/watch?v=0hyRw7zeVcs) about what not to do in services.

So, I hope this article clarified to you what are Service classes and how to use them in most typical ways.
