# Laravel Roles and Permissions: Middleware, Gates or Policies?
When creating an application, you will need some restrictions for your users. Laravel offers a variety of ways how to implement this. In this tutorial, I will show you four examples:

- Simple Middleware
- Restriction with Gates
- From Gates to Policies
- Roles in DB with Model
There are also well-known packages like spatie/laravel-permission, but for the purpose of this article, I deliberately want to show what Laravel offers in its core, without external packages.

## Scenario Setup
In this example, we will work with Users and Tasks and allow different users to access different pages related to the tasks.

Here's our setup:

**Migrations**
```
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->boolean('is_admin')->default(0);
    $table->rememberToken();
    $table->timestamps();
});
 
Schema::create('tasks', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->string('name');
    $table->date('due_date');
    $table->timestamps();
});
```
Now, let's define the relationships:

*app/Models/User.php*
```
use App\Models\Task;
 
// ...
 
public function tasks()
{
    return $this->hasMany(Task::class);
}
```
*app/Models/Task.php*
```
use App\Models\User;
 
// ...
 
public function user()
{
    return $this->belongsTo(User::class);
}
```
## Example 1. Middleware: Different Pages by User Role
In this example, we'll separate our routes/models and controllers by user role. It means that we will have two pages - one for the simple user and one for the admin, and we will restrict it with Middleware.

So, we generate this Middleware class:
```
php artisan make:middleware IsAdmin
```
*app/Http/Middleware/IsAdmin.php*
```
public function handle(Request $request, Closure $next): Response
{
    if (!auth()->check() || !auth()->user()->is_admin) {
        abort(403);
    }
    return $next($request);
}
```
As you can see, we just have the field users.is_admin in the DB and filter by that.

Next, we need to register the Middleware and assign it a name:

*app/Http/Kernel.php*
```
protected $middlewareAliases = [
    // ...
    'is_admin' => App\Http\Middleware\IsAdminMiddleware::class
];
```
Finally, we can use it in our routes.

*routes/web.php*
```
Route::middleware('is_admin')->prefix('admin')->group(function () {
    Route::get('/tasks', [\App\Http\Controllers\Admin\TasksController::class, 'index']);
});
Route::prefix('user')->group(function () {
    Route::get('/tasks', [\App\Http\Controllers\MyTaskController::class, 'index']);
});
```
As you can see, the first route is protected with middleware('is_admin').

For example, Controllers could look like this: simple users see only their tasks, whereas admin sees all the tasks.

*app/Http/Controllers/MyTaskController.php*
```
public function index()
{
    $tasks = Task::where('user_id', auth()->id())->orderBy('due_date')->get();
 
    return view('tasks.index', compact('tasks'));
}
```
*app/Http/Controllers/Admin/TasksController.php*
```
public function index()
{
    $tasks = Task::with(['user'])->orderBy('due_date')->get();
 
    return view('admin.tasks.index', compact('tasks'));
}
```
So, you can restrict some pages/endpoint/functionality just with Middleware, without any Gates/Policies/Roles. But if you want to go a bit deeper...

## Example 2. Gates and Policies
With this example, we'll look into using Gates and Policies to define Permissions for our users.

**Notice:** the term "Gates" in Laravel context is actually almost the same as "Permissions", it defines whether someone has access to perform certain action or feature.

First, you define the Gates globally, in any Service Provider, like AuthServiceProvider.

*app/Providers/AuthServiceProvider.php*
```
use Illuminate\Support\Facades\Gate;
 
// ...
 
public function boot(): void
{
    Gate::define('create_task', fn($user) => $user->is_admin);
    Gate::define('edit_task', fn($user) => $user->is_admin);
    Gate::define('delete_task', fn($user) => $user->is_admin);
}
```
Then, we can use them in our views with @can directive:

*resources/views/tasks/index.blade.php*
```
{{-- ... --}}
@can('create_task')
    <a href="{{ route('tasks.create') }}"
       class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded inline-block">Create new task</a>
@endcan
 
{{-- ... --}}
 
@can('edit_task')
    <a href="{{ route('tasks.edit', $task) }}"
       class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded inline-block">Edit</a>
@endcan
@can('delete_task')
    <form action="{{ route('tasks.destroy', $task) }}" method="POST" class="inline-block">
        @csrf
        @method('DELETE')
        <button type="submit"
                class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded inline-block">Delete
        </button>
    </form>
@endcan
```
As you can see, we've used the create_task, edit_task, and delete_task permission check from our defined Gates in AuthServiceProvider.

Admin will see buttons on the screen:

![image](https://user-images.githubusercontent.com/11309713/235358641-a776c776-487d-4df1-a119-1e52098daee5.png)

And user will not see any buttons:

![image](https://user-images.githubusercontent.com/11309713/235358646-6397b0fb-906d-4ec9-9690-4b3cc1013467.png)

But we've protected only the front-end. What if someone knows the URL so they could bypass the button?

Let's take one more step to ensure that users can't access our Controller methods. Here's an example of a full CRUD:

*app/Http/Controllers/TaskController.php*
```
public function create()
{
    $this->authorize('create_task');
 
    return view('tasks.create');
}

public function store(StoreTaskRequest $request)
{
    $this->authorize('create_task');
 
    Task::create($request->validated());
 
    return redirect()->route('tasks.index');
}
 
public function edit(Task $task)
{
    $this->authorize('edit_task');
}
 
public function update(UpdateTaskRequest $request, Task $task)
{
    $this->authorize('edit_task');
 
    $task->update($request->validated());
 
    return redirect()->route('tasks.index');
}
 
public function destroy(Task $task)
{
    $this->authorize('delete_task');
 
    $task->delete();
 
    return redirect()->route('tasks.index');
}
```
## Example 3. From Gates to Policies
If you want a more structured approach around one Model, instead of creating separate create_task, edit_task and xxxxx_task Gates, you may combine them into Policies.

Let's create a new Policy:
```
php artisan make:policy TaskPolicy
```
*app/Policies/TaskPolicy.php*
```
use App\Models\Task;
 
class TaskPolicy
{
    public function create(User $user)
    {
        return $user->is_admin;
    }
 
    public function update(User $user, Task $task)
    {
        return $user->is_admin || $user->id === $task->user_id;
    }
 
    public function delete(User $user)
    {
        return $user->is_admin;
    }
}
```
Next, register it in the AuthServiceProvider:

*app/Providers/AuthServiceProvider.php*

 ```
use App\Models\Task;
use App\Policies\TaskPolicy;
 
// ...
 
protected $policies = [
    // ...
    Task::class => TaskPolicy::class,
];
```
Then update our views to use the new Policy instead of Gates:

*resources/views/tasks/index.blade.php*
```
{{-- ... --}}
@can('create', \App\Models\Task::class)
    <a href="{{ route('tasks.create') }}"
       class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded inline-block">Create new task</a>
@endcan
 
{{-- ... --}}
 
@can('update', $task)
    <a href="{{ route('tasks.edit', $task) }}"
       class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded inline-block">Edit</a>
@endcan
@can('delete', \App\Models\Task::class)
    <form action="{{ route('tasks.destroy', $task) }}" method="POST" class="inline-block">
        @csrf
        @method('DELETE')
        <button type="submit"
                class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded inline-block">Delete
        </button>
    </form>
@endcan
```
A few things to note here:

1. Our @can now has a class next to it to indicate which Policy we are using.
2. Since our @can('update') will check for ownership - we are passing a second parameter to it - the task to check if it's owned by the user @can('update', $task)
After these changes, our admin view will not change. On the other hand, our simple user will have an Edit button available to them:

![image](https://user-images.githubusercontent.com/11309713/235358756-79da46c5-807d-4fe8-bc10-ef50b416b70c.png)

Similarly to Gates, we also need to protect the back-end with Policies.

*app/Http/Controllers/TaskController.php*
```
public function create()
{
    $this->authorize('create', \App\Models\Task::class);
 
    return view('tasks.create');
}
 
public function store(StoreTaskRequest $request)
{
    $this->authorize('create', \App\Models\Task::class);
 
    Task::create($request->validated());
 
    return redirect()->route('tasks.index');
}
 
public function edit(Task $task)
{
    $this->authorize('update', $task);
}
 
public function update(UpdateTaskRequest $request, Task $task)
{
    $this->authorize('update', $task);
 
    $task->update($request->validated());
 
    return redirect()->route('tasks.index');
}
 
public function destroy(Task $task)
{
    $this->authorize('delete', \App\Models\Task::class);
 
    $task->delete();
 
    return redirect()->route('tasks.index');
}
```
Be aware that we need to pass the Task model to the authorize method to tell the code which Policy we are using. And for editing tasks - we will pass a specific Model instance, to check if that task is owned by specific user.

## Example 4. Adding Roles DB Table
Sometimes just a simple difference between users and admins is not enough. You might want to add a manager Role to your system. Then it becomes more complicated than just users.is_admin and requires a separate DB structure.

First, we need to add a new DB table for Roles:

**Migration**
```
Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
});
```
Then, add a Role relationship to the User:

**Migration**
```
Schema::table('users', function (Blueprint $table) {
    $table->foreignId('role_id')->nullable()->constrained();
});
```
**app/Models/User.php**
```
// ...
public function role()
{
    return $this->belongsTo(Role::class);
}
```
It will allow us to use $user->role_id to get the Role of the user by the ID quickly:

![image](https://user-images.githubusercontent.com/11309713/235358813-b17a21cd-ab4c-4256-a79d-fc30313928a2.png)

Then, we create our Roles, it may be done in a Seeder or manually:
```
Role::create(['name' => 'Admin']);
Role::create(['name' => 'User']);
Role::create(['name' => 'Manager']);
```
We could already modify our Policy to use the role_id, but for better readability of the code, let's assign those IDs to the constants in our Role model:

*app/Models/Role.php*
```
// ...
 
public const ADMIN = '1';
public const USER = '2';
public const MANAGER = '3';
```
This way we will not have to use 1 as an admin but we will have Role::ADMIN that is much more readable, see below in the Policy:

*app/Policies/TaskPolicy.php*
```
public function create(User $user)
{
    return in_array($user->role_id, [Role::ADMIN]);
}
 
public function update(User $user, ?Task $task)
{
    return in_array($user->role_id, [Role::ADMIN]) || ($task && $user->id === $task->user_id);
}
 
public function delete(User $user)
{
    return in_array($user->role_id, [Role::ADMIN]);
}
```
After this modification - our code should work as previously. But we also can allow our Manager to create tasks. All we have to do is add the Manager to our Policy:

*app/Policies/TaskPolicy.php*
```
public function create(User $user)
{
    return in_array($user->role_id, [Role::ADMIN, Role::MANAGER]);
}
```
That's it. If we assign the manager Role to a User he will see a create button:



With this example, you can expand your code control with as many roles as you need and define those permissions individually.
