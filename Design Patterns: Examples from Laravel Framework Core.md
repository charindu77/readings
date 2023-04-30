## Design Patterns: Examples from Laravel Framework Core
Wanna learn design patterns? Here's a "secret": you've all actually USED them already while working with Laravel itself. Let's see the examples of patterns like Facade, Builder, and Adapter in the framework code.

## Builder Pattern
The most common and familiar Builder pattern is the Query Builder in Laravel. Of course, it is extended by Eloquent Models but the core example is still the Query Builder. Let's look at the example:

Let's get a basic example:

### Query Builder
```
DB::table('users')
    ->where('name', 'John')
    ->where('age', '>', 18)
    ->orderBy('age')
    ->get();
```
So what did we do, and why is it a Builder? We've built a query by chaining methods on top of a base. So what does it do under the hood? Let's look at the example:

- DB::table('users') - We indicate that we will work with the users table
- ->where('name', 'John') - Added a condition that only John users should be selected
- ->where('age', '>', 18) - Added a condition that only users with an age greater than 18 should be selected
- ->orderBy('age') - Order the users by age
- ->get() - Executed the query and retrieved the results from the database
Each of these individual blocks will append to a raw SQL query as we are building it in parts. Think about it like this:

- DB::table('users') - SELECT * FROM users...
- ->where('name', 'John') - ...WHERE name = 'John'...
- ->where('age', '>', 18) - ...WHERE name = 'John' AND age > 18...
- ->orderBy('age') - ...ORDER BY age...
- ->get() - SELECT * FROM users WHERE name = 'John' AND age > 18 ORDER BY age
If we want to look at how we are building an update query we can look at this example:

### Query Builder
```
DB::table('users')
    ->where('name', 'John')
    ->update(['age' => 18]);
```
This will do the following:

- DB::table('users') - We indicate that we will work with the users table
- ->where('name', 'John') - Added a condition that only John users should be updated
- ->update(['age' => 18]) - Updated the age column to be 18 for all users that match the condition
Each of those is individually added to the query. Let's look under the hood:

- DB::table('users') - UPDATE users...
- ->where('name', 'John') - ...WHERE name = 'John'...
- ->update(['age' => 18]) - ...SET age = 18...
- ->get() - UPDATE users SET age = 18 WHERE name = 'John'
As you can see, we are building the query in parts and then executing it. This is the Builder pattern in action!

## Facade Pattern
Another commonly used pattern within your code will be Facade Pattern. It's a really simple pattern that allows you to hide the complexity of a system behind a simple interface. Let's look at the example:

Let's look at the example of the Auth Facade:

### Auth Facade
```
Auth::login($user);
Auth::check();
Auth::user();
Auth::id();
Auth::logout();
Auth::attempt($credentials);
```
I think that a lot of you are familiar with this Facade and used it multiple times in your application. If you think that you have never used it, there's another way that Laravel implements this - via helper:

### Auth helper
```
auth()->login($user);
auth()->check();
auth()->user();
auth()->id();
auth()->logout();
auth()->attempt($credentials);
```
Both of these examples are using the same Facade but different syntax to help you out. You have the full Facade class Auth but also a helper auth() that will return the same instance of the Facade. So what's under it?

Under the Auth, we have a full class: \Illuminate\Auth\AuthManager that is responsible for managing the authentication. There are a lot of methods there, but you don't have to worry about the majority of them for your application so for your convenience the Facade provides the most commonly used methods. This is the Facade pattern in action!

**Bonus** - it also provides one custom method:
```
Illuminate/Auth/AuthManager

/**
 * Register the typical authentication routes for an application.
 *
 * @param  array  $options
 * @return void
 *
 * @throws \RuntimeException
 */
public static function routes(array $options = [])
{
    if (! static::$app->providerIsLoaded(UiServiceProvider::class)) {
        throw new RuntimeException('In order to use the Auth::routes() method, please install the laravel/ui package.');
    }
 
    static::$app->make('router')->auth($options);
}
```
Which will register the authentication routes for you.

## Adapter Pattern
Working with different systems can be hard as every system might have a different standard. It doesn't matter if it's a 3rd party API or email service - they all have different standards and expect different data. This is where Adapter pattern comes in handy! Its main goal is to take whichever format you have and adapt it to the required format to become compatible. Think about it like using a power adapter for your phone to convert the wall socket into a USB cable. Did you know that you've been using it without fully understanding what it does? They are commonly used as Notifications in Laravel. Let's look at them:

Notifications in Laravel adapt your data to interact with different drivers such as email, broadcasting channels, Slack, and many more with custom drivers. So how does it work? Let's take a look at the examples:

Let's look at the toMail method that we usually use:

### Notification
```
public function toMail($notifiable)
{
    return (new MailMessage)
        ->line('The introduction to the notification.')
        ->action('Link button', route('link'));
}
```
In this example, we've transformed our data to be compatible with the MailMessage class which will be used to send an email. This is the adapter pattern in action! There are two layers to it:

1. Us transforming our data to be compatible with the MailMessage class
2. The MailMessage class transforming our data to be compatible with the email API
And all of that is done seamlessly without us even knowing about it! We don't have to worry about the email driver or what data it expects. We just have to transform our data to be compatible with the MailMessage class and that's it!

What if we want more drivers? No problem! We can just add more methods to our notification class (e.g. toSlack) and transform our data to be compatible with the SlackMessage class. Let's look at the example:

### Notification
```
public function toMail($notifiable)
{
    return (new MailMessage)
        ->line('The introduction to the notification.')
        ->action('Link button', route('link'));
}
 
public function toSlack($notifiable)
{
    return (new SlackMessage)
        ->content('The introduction to the notification.');
}
```
And once again, there are going to be a few layers of the adapter pattern:

1. Us transforming our data to be compatible with the MailMessage class
2. The MailMessage class transforming our data to be compatible with the email API
3. Us transforming our data to be compatible with the SlackMessage class
4. The SlackMessage class transforming our data to be compatible with the Slack API
While we still have a nice developer experience!

## Singleton Pattern
Another pattern that we use quite extensively under the hood is the Singleton pattern. Its main goal is to have a single instance of a specific class that's always accessed no matter where you are in your application or how many times you call it. It solves the problem of re-creating the same instance over and over again. Let's look at the example:

In Laravel, we have quite a few singletons that we use but one stands out as a really common - Request and request() usage. Let's look at the example:

In this case, to demonstrate that it is still the same instance being accessed we'll use both the Request class and request() helper. We'll set fake data on one of them and then dump the other one to see if it's the same data. Let's look at the example:

### Controller
```
public function index(Request $request)
{
    dump(request()->all());
    request()->merge(['test' => '123']);
    dump(request()->all());
    dd($request->all());
}
```
Running this in the browser will show us the following:

![image](https://user-images.githubusercontent.com/11309713/235370166-8de00dc0-7262-4d43-b4df-24b48ec46ee8.png)

So what happened here?

1. We've checked initially to see what our request data is
2. We've added a new key to the request data using the request() helper
3. We've checked that our request() helper has the new key
4. We've attempted to check that our Request class has the new key
All the data was in place for both the request() and Request classes. It is because they are using the same instance of the Request class. This is the Singleton pattern in action!

## Observer Pattern
Another common pattern in Laravel is the Observer pattern. Its main goal is to track events that are happening and notify other classes about them. This is done with the help of the Observers that are listening for specific events on the Models. Let's look at the example:

app/Observers/TransactionObserver.php
```
// ...
class TransactionObserver
{
    public function created(Transaction $transaction): void
    {
        $transaction->client->notify(new \App\Notifications\TransactionCreated($transaction));
    }
    // ...
}
```
app/Models/Transaction.php
```
use App\Observers\TransactionObserver;
 
// ...
 
public function boot(): void
{
    self::observe(TransactionObserver::class);
}
```
We've just created an Observer that will track the created event on the Transaction model and notify the client about it with a custom notification. Do you need to also generate a PDF invoice? No problem! Just add another method to the observer, and you're good to go!

app/Observers/TransactionObserver.php
```
use App\Services\TransactionService;
 
// ...
 
public function created(Transaction $transaction): void
{
    $transactionService = new TransactionService();
    $transactionService->generateInvoice($transaction);
 
    $transaction->client->notify(new \App\Notifications\TransactionCreated($transaction));
}
```
That's it, we've added more functionality to our Observer. Once an event is triggered it will now generate a PDF and notify the client. This is the Observer pattern in action!

## Conclusion
Patterns are common within all the tools that we have available to us. We've just taken a glance at what you might find commonly used but there are many more patterns applied in Laravel. I hope that this article will help you understand them better and maybe even use them in your projects!
