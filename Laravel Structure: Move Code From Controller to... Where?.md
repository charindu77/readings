# Laravel Structure: Move Code From Controller to... Where?
One of the most common questions I see about Laravel is how to structure the project. Or, in other words, where to put the logic out of the Controllers? In this article, I will try to show the options, trying to shorten one Controller method as an example.

This is a text-form tutorial based on the section of my video course How To Structure Laravel Projects

We will discuss how to move the logic from the Controller, to...:

- Form Requests
- Eloquent Mutators
- Eloquent Observers
- Service Classes
- Action Classes
- Jobs
- Events and Listeners
- Global Helpers
- Traits
- Base Controllers
- Repository Classes
A lot to cover, huh? So let's get started.

## Initial Controller code
Before starting to cleanup the Controller, here's the code which we will try to make shorter:
```
public function store(Request $request)
{
    $this->authorize('user_create');
 
    $userData = $request->validate([
        'name' => 'required',
        'email' => 'required|unique:users',
        'password' => 'required',
    ]);
 
    $userData['start_at'] = Carbon::createFromFormat('m/d/Y', $request->start_at)->format('Y-m-d');
    $userData['password'] = bcrypt($request->password);
 
    $user = User::create($userData);
    $user->roles()->sync($request->input('roles', []));
 
    Project::create([
        'user_id' => $user->id,
        'name' => 'Demo project 1',
    ]);
    Category::create([
        'user_id' => $user->id,
        'name' => 'Demo category 1',
    ]);
    Category::create([
        'user_id' => $user->id,
        'name' => 'Demo category 2',
    ]);
 
    MonthlyReport::where('month', now()->format('Y-m'))->increment('users_count');
    $user->sendEmailVerificationNotification();
 
    $admins = User::where('is_admin', 1)->get();
    Notification::send($admins, new AdminNewUserNotification($user));
 
    return response()->json([
        'result' => 'success',
        'data' => $user,
    ], 200);
}
```
Quite a big method, right? Now, let's walk through the options to shorten it.

**Notice:** at the end of the day, it's your personal preference where to move the code, you MAY choose any option listed below.

## Validation to Form Request
We will start by extracting validation into Form Request. In this example, validation is simple, with three fields, but in real life, you could have 10+ fields.

Actually, we have two parts of the validation:

- permissions
- input data validation
Both of them CAN be moved to the Form Request class.

Let's start by creating a Form Request:
```
php artisan make:request StoreUserRequest
```
Now we have the app\Http\Requests\StoreUserRequest.php file which has two methods inside: authorize() for permissions and rules() for data validation. So, Form Request would look like this:

#### app\Http\Requests\StoreUserRequest.php:
```
class StoreUserRequest extends FormRequest
{
    public function authorize()
    {
        return Gate::allows('user_create');
    }
 
    public function rules()
    {
        return [
            'name' => 'required',
            'email' => 'required|unique:users',
            'password' => 'required',
        ];
    }
}
```
In the Controller, instead of the default Request class, we need to inject our StoreUserRequest, and validated data can be accessed using the validated() method from the Request. Now, the Controller will look like this:
```
public function store(StoreUserRequest $request)
{
    $userData = $request->validated();
 
    $userData['start_at'] = Carbon::createFromFormat('m/d/Y', $request->start_at)->format('Y-m-d');
    $userData['password'] = bcrypt($request->password);
 
    $user = User::create($userData);
    $user->roles()->sync($request->input('roles', []));
    //
}
```
A few first lines of the Controller shortened. Let's move on.

## Data Change to Mutators or Observers
Let's say that you want to transform some data before saving it into the database.

Two examples here formatting the date and encrypting the password.

Now, we do it in the Controller, but let's use Laravel Eloquent features for that. I will show you two methods, one using Mutators and the another using Observers.

- Mutators
In Eloquent models, you can define Mutators. There are two ways how you can define them, the "old" way, and the "new" way. Below are examples in both ways:

Laravel 9 and below:
```
public function setStartAtAttribute($value)
{
    $this->attributes['start_at'] = Carbon::createFromFormat('m/d/Y', $value)->format('Y-m-d');
}
 
public function setPasswordAttribute($value)
{
    $this->attributes['password'] = bcrypt($value);
}
```
Since Laravel 9:
```
protected function startAt(): Attribute
{
    return Attribute::make(
        set: fn ($value) => Carbon::createFromFormat('m/d/Y', $value)->format('Y-m-d');
    )
}
 
protected function password(): Attribute
{
    return Attribute::make(
        set: fn ($value) => bcrypt($value));
    )
}
```
- Observers
You can create the Observer by running command:
```
php artisan make:observer UserObserver --model=User
```
If you will open app/Observers/UserObserver.php our created observer, you will see there are generated methods about events that already happened like created() or updated(). But you can define creating() which will be called before creating a record.

#### app/Observers/UserObserver.php:
```
class UserObserver
{
    public function creating(User $user)
    {
        $user->start_at = Carbon::createFromFormat('m/d/Y', $user->start_at)->format('Y-m-d');
        $user->password = bcrypt($user->password);
    }
}
```
But this method isn't mentioned in Laravel documentation so probably it's not officially recommended. So if you do want to shorten the controller and move that logic somewhere I probably would recommend using Mutators.

So now, we don't need those two lines in the Controller, and we don't need the $userData variable, we can pass validated data directly into the User create method.
```
public function store(StoreUserRequest $request)
{
    $user = User::create($request->validated());
    $user->roles()->sync($request->input('roles', []));
    //
}
```
## Adding a Service Class
We continue transforming our Controller method and moving logic elsewhere. Now we get to lines which are probably the main logic of saving the data in the DB. For this, we will create a Service class.

There's no make:service Artisan command, so manually create a PHP app/Services/UserService.php file which would look like that:
```
namespace App\Services;
 
class UserService
{
    public function create(array $userData): User
    {
        $user = User::create($userData);
        $user->roles()->sync($userData['roles']);
 
        return $user;
    }
}
```
Here we make the create() method which accepts an array of validated data. In this method, we create a user and sync roles, and then return the created user.

Now, how to call this service in the controller? There are at least two ways.

The first one is to initialize the service by doing the (new UserService())->create($request->validated()) and passing validated data to the create() method.

The second is injecting into a method, our case store(), type-hinting that and assigning a variable. So now our controller would like this:
```
public function store(StoreUserRequest $request, UserService $userService)
{
    $user = $userService->create($request->validated());
    //
}
```
If you want to find out how that type-hinting magic works, I have this article: Laravel Service Container: What Beginners Need to Know

## Service into Action: What's the Difference?
Another option instead of a Service class, is called Action class. Again, there's no make:action Artisan command, so manually create a file app/Actions/CreateUserAction.php. Inside, typically there's one method called handle() or execute().
```
namespace App\Actions;
 
class CreateUserAction
{
    public function execute(array $userData): User
    {
        $user = User::create($userData);
        $user->roles()->sync($userData['roles']);
 
        return $user;
    }
}
```
To call this Action class in the Controller, you would just initialize the action and call the execute() method by passing data to it.
```
public function store(StoreUserRequest $request)
{
    $user = (new CreateUserAction())->execute($request->validated());
    //
}
```
As you can see, there's not much difference when using a Service class and an Action class. The difference is more like do you divide your logic:

- Either into model-related entities like UserService or TaskService, with many methods inside
- Or, each operation as an Action class like CreateUserAction or UpdateUserAction, with one method
Of course, inside of that Action class, you may also have some private methods for more logic, if you have something more complicated.

## Jobs and Queues
Imagine there is some kind of sequence of actions to be performed "in the background", in addition to creating the main logic. For example, after user registration, you want to prepare some demo data for the dashboard. You may want to put it into the background queue, so the user wouldn't wait for the operation to finish.

Jobs are very similar to Actions, but Jobs may be put in a Queue. And they have Artisan command to create them:
```
php artisan make:job NewUserDataJob
```
And our job would be like this:
```
class NewUserDataJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
 
    public function __construct(public User $user)
    {}
 
    public function handle()
    {
        Project::create([
            'user_id' => $this->user->id,
            'name' => 'Demo project 1',
        ]);
        Category::create([
            'user_id' => $this->user->id,
            'name' => 'Demo category 1',
        ]);
        Category::create([
            'user_id' => $this->user->id,
            'name' => 'Demo category 2',
        ]);
    }
}
```
All that's left is to dispatch the Job in the Controller:
```
public function store(StoreUserRequest $request)
{
    $user = (new CreateUserAction())->execute($request->validated());
    NewUserDataJob::dispatch($user);
    //
}
```
And then, you need to separately configure everything around the queue. If you want to learn about that, I have a separate course Queues in Laravel

## Events and Listeners
Another possible option for "tasks in the background" is to call the Event in the Controller, and allow different classes (current ones or the future ones) to "listen" to that event.

Imagine the scenario in which you need to inform some other classes that the new user is registered. For example, we want to update a Monthly Report. So first, let's make an Event class:
```
php artisan make:event NewUserRegistered
```
And a Listener:
```
php artisan make:listener MonthlyReportNewUserListener
```
Now we can dispatch the Event in the Controller, similar to a job:
```
public function store(StoreUserRequest $request)
{
    $user = (new CreateUserAction())->execute($request->validated());
    NewUserDataJob::dispatch($user);
 
    NewUserRegistered::dispatch($user); 
    //
}
```
Inside the Event, we need to accept a User, so every Listener class would have access to that parameter:
```
class NewUserRegistered
{
    public function __construct(public User $user)
    {}
}
```
And then in the EventServiceProvider, we register our Event should be listened to:
```
class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
        NewUserRegistered::class => [ 
            MonthlyReportNewUserListener::class, 
        ] 
    ];
}
```
Inside the MonthlyReportNewUserListener listener class, we have an $event parameter in the handle() method. Inside that we move the code from the Controller:
```
class MonthlyReportNewUserListener
{
    public function handle(NewUserRegistered $event)
    {
        MonthlyReport::where('month', now()->format('Y-m'))->increment('users_count');
    }
}
```
Another example of event-listeners come from Laravel itself.

In the Controller, we don't need to send email verification notifications, because it is already handled by Laravel's Registered event and SendEmailVerificationNotification listener.

But we can create another Listener to send more notifications: for example, notify admins about something.

First, create a Listener and register it in the EventServiceProvider:
```
php artisan make:listener NewUserSendAdminNotifications
```
#### app/Providers/EventServiceProvider.php:
```
class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
        NewUserRegistered::class => [
            MonthlyReportNewUserListener::class,
            NewUserSendAdminNotifications::class, 
        ]
    ];
}
```
Now in the NewUserSendAdminNotifications listener, in the handle() method, move the code from the Controller. And you can access the User from the $event using $user->event:
```
class NewUserSendAdminNotifications
{
    public function handle(NewUserRegistered $event)
    {
        $admins = User::where('is_admin', 1)->get();
        Notification::send($admins, new AdminNewUserNotification($event->user));
    }
}
```
So now, the full Controller looks just like this, from 37 to just 10 lines of code:
```
public function store(StoreUserRequest $request)
{
    $user = (new CreateUserAction())->execute($request->validated());
 
    NewUserDataJob::dispatch($user);
 
    NewUserRegistered::dispatch($user);
 
    return response()->json([
        'result' => 'success',
        'data' => $user,
    ], 200);
}
```
Additionally, let's talk about a few more options that we didn't directly use here.

## Global Helpers
Helper classes have been around for a while. It's any Class that has some helper methods related to some topic. For example, DateHelper, or CurrencyHelper. For example, earlier we created the Attribute startAt in the User model, which has some data manipulation that can be added to a helper.
```
protected function startAt(): Attribute
{
    return Attribute::make(
        set: fn ($value) => Carbon::createFromFormat('m/d/Y', $value)->format('Y-m-d');
    )
}
```
Create a new file app/Helpers/DateHelper.php (there's no Artisan command for this), and inside the DateHelper class, add the convertToDB() method which will have the code from the attribute.
```
namespace App\Helpers;
 
class DateHelper
{
    public static function convertToDB($date)
    {
        return Carbon::createFromFormat('m/d/Y', $date)->format('Y-m-d');
    }
}
```
Now you can change Attribute to use this helper:
```
protected function startAt(): Attribute
{
    return Attribute::make(
        set: fn ($value) => DateHelper::convertToDB($value);
    )
}
```
And whenever you need to convert the date from the same format before saving it to DB, now you can just use this helper.

## Repeating parts: Traits or Base Controller?
Returning the result may be a repeating part of various controllers, especially in API controllers.

One option is to create a separate logic for responses from API. There are at least two ways how you can do that.

First, you can create the logic in the Base Controller which is app/Http/Controllers/Controller.php:
```
class Controller extends BaseController
{
    use AuthorizesRequests, DispatchesJobs, ValidatesRequests;
 
    public function respondOk($data)
    {
        return response()->json([
            'result' => 'success',
            'data' => $data,
        ], 200);
    }
}
```
And then in your "regular" Controller, the return would be changed to $this->respondOk().
```
public function store(StoreUserRequest $request)
{
    $user = (new CreateUserAction())->execute($request->validated());
 
    NewUserDataJob::dispatch($user);
    NewUserRegistered::dispatch($user);
 
    return response()->json([ 
        'result' => 'success',
        'data' => $user,
    ], 200);
    $this->respondOk($user); 
}
```
Another option is to use Traits. For example, make the app/Traits/APIResponsesTrait.php file and inside this trait create the same method respondOk():
```
namespace App\Traits;
 
trait APIResponsesTrait
{
    public function respondOk($data)
    {
        return response()->json([
            'result' => 'success',
            'data' => $data,
        ], 200);
    }
}
```
Now you just need to add this Trait either to each controller or the Base Controller.
```
class Controller extends BaseController
{
    use AuthorizesRequests, DispatchesJobs, ValidatesRequests;
    use APIResponsesTrait; 
}
```
After adding a trait, you can use $this->respondOk() method the same way.

## Why not Repository Class?
Similarly to the Services and Actions, one of the options is moving logic to Repositories.

It was a very popular pattern in Laravel 4 and 5 days. But times have changed and not many people use it now.

Repository idea has been an extra layer on top of Eloquent, between Eloquent and Controller. But the idea of repositories comes from general programming theory that a repository is a layer between the Controller and the Database. It makes sense when you use programming languages or frameworks that doesn't have an Eloquent ORM mechanism.

In the case of Laravel, Eloquent itself is the layer between the Controller and Database, so it kinda acts as Repository Pattern.

Instead of
```
SELECT * FROM USERS;
```
you do
```
User::all();
```
That's why adding a Repository as another layer on top of already a repository-like layer doesn't make much sense, in my opinion, and doesn't give many benefits.

That's it in this tutorial. Again, I will repeat: you are free to choose whichever of the options or patterns I mentioned above, there are no strict rules. Your goal is for the code to be easily readable and maintainable by future developers, including yourself.

If you want more examples of Laravel structure, join my 2-hour course How to Structure Laravel Projects.
