# Eloquent Foreign Keys: 3 Common Mistakes to Avoid
Adding foreign keys can sometimes be tricky. You might get an error message or see that it doesn't work as expected.

There are 3 common mistakes that we see developers make when adding foreign keys to their databases:

- Forgetting to add Constraints
- Adding new Foreign Keys Without Proper Default Data
- Allowing to Delete Parent Records when Child Record Exists
Let's see those mistakes and how to fix them!

## Mistake 1: Forgetting to add Constraints
Database constraints are a valuable feature we should not ignore. They help us protect from missing related records in our database.

Let's build a basic example here:

**Migration**
```
// Create Categories table
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('category')->nullable();
    $table->timestamps();
});
 
// Create a child Questions table with a relationship to the parent Categories table
Schema::create('questions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id');
    $table->longText('question')->nullable();
    $table->longText('answer')->nullable();
    $table->timestamps();
});
```
You might think that we are good to go here. However, we are not. It allows us to create new Questions with a category that doesn't exist:
```
$question = Question::create([
    'category_id' => 155,
    'question' => 'How to use Laravel?',
    'answer' => 'You can use Laravel by following the documentation',
]);
```
And this will work perfectly fine. Why? Because we don't tell our database to check for the category id.

![image](https://user-images.githubusercontent.com/11309713/235356720-7824da2b-9842-4b8c-8b47-94bd933a2789.png)

We have 0 categories in our database, but we can still create a new Question with category id 155. It is not good. Let's fix it!

**Migration**
```
// ...
 
// Create a child Questions table with a relationship to the parent  Categories table
Schema::create('questions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained();
    $table->longText('question')->nullable();
    $table->longText('answer')->nullable();
    $table->timestamps();
});
```
When we try to run the same code of Question::create(), we get an error:

![image](https://user-images.githubusercontent.com/11309713/235356736-03a50438-afda-4699-9e41-c3855025d73a.png)

It prevents us from creating resources linked with something that doesn't exist.

## Mistake 2: Adding new Foreign Keys Without Proper Default Data
Adding new relationships can be challenging. We can prevent this in a couple of ways:

Our base migration setup looks like this:

**Migration**
```
// Create Categories table
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('category')->nullable();
    $table->timestamps();
});
```
And make sure we have some data in our database:
```
$category = Category::create([
    'category' => 'General'
]);
```
Next we'll try to add a new foreign key to our categories to add additional support for "subcategories":

**Migration**
```
Schema::table('categories', function (Blueprint $table) {
    $table->foreignId('category_id')->constrained();
});
```
Now php artisan migrate will show this error:

![image](https://user-images.githubusercontent.com/11309713/235356810-50d90a9e-6f7e-4f1c-b88a-f936b9aba88d.png)

Because we are trying to add a new foreign key to a table that already has data inside we need to add a default OR to make the column nullable:

## Potential Fix 1. Using nullable column
One way to fix this is to make the column nullable:

**Migration**
```
Schema::table('categories', function (Blueprint $table) {
    $table->foreignId('category_id')->nullable()->constrained();
});
```
Running the migration now should work just fine:

![image](https://user-images.githubusercontent.com/11309713/235356838-abfafc9a-8671-448c-88d8-5469b3ea941a.png)

## Potential Fix 2. Using default value
Another way to fix this is to add a default value. Since we have data already in the table, we can use it to set a default value:

**Migration**
```
Schema::table('categories', function (Blueprint $table) {
    $table->foreignId('category_id')->default(1)->constrained();
});

```

And in our database we can see that the default value has been set:

![image](https://user-images.githubusercontent.com/11309713/235356860-359bf878-a612-473e-9d9f-4040ff834bc8.png)

## Mistake 3: Allowing to Delete Parent Records when Child Records Exist
Once you add a foreign key to the model, you can no longer delete the parent record if there are any child records.

Let's build a basic example here:

**Migration**
```
// Create  Categories table
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('category')->nullable();
    $table->timestamps();
});
 
// Create a child Questions table with a relationship to the parent  Categories table
Schema::create('questions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained();
    $table->longText('question')->nullable();
    $table->longText('answer')->nullable();
    $table->timestamps();
});
```
And once we attempt to delete our parent record, we get an error:
```
$category = Category::find(1);
$category->delete();
```
![image](https://user-images.githubusercontent.com/11309713/235356899-dee51f68-1be4-4c81-b96c-a01db1c8470b.png)

Why is that? Our resources are constrained, and deleting Category would mean that the Questions has no parent Category left. There are a couple of ways we can avoid it:

- Cascading deletion
- Restricting deletion
Let's dive into those options.

## Potential Fix 1. Cascading deletion
The first solution might be cascading the records in the database. It means that once you delete a parent record the database engine will delete all the related child records for you.

To set this up, we need to modify our migration:

**Migration**
```
Schema::create('questions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained()->cascadeOnDelete();
    $table->longText('question')->nullable();
    $table->longText('answer')->nullable();
    $table->timestamps();
});
```
After this simple change in our migration - we will avoid getting an error when deleting our Category model.

Notice about SoftDeletes. Cascading with Soft Delete does not work. However, there's a way to achieve this. We can use the deleting event on our model:

*app/Models/Category.php*
```
protected static function boot()
{
    parent::boot();
 
    static::deleting(function ($category) {
        $category->questions()->delete();
    });
}
```
Alternatively, you can use a package called Laravel Cascade Soft Deletes.

## Potential Fix 2. Restricting deletion
Another common way is to restrict the deletion. We can do it in a few ways:

Preventing deletion in the controller and warning users about it
*app/Http/Admin/CategoryController.php*
```
public function destroy(Category $category)
{
    if ($category->questions()->count() > 0) {
        return back()->with('error', 'You can not delete this category because it has questions');
    }
 
    $category->delete();
 
    return back();
}
```
To display the message we should add some code to our view:

*resources/views/admin/categories/index.blade.php*
```
{{-- ... --}}
 
@if(session()->has('error'))
    <div class="alert alert-danger">
        {{ session()->get('error') }}
    </div>
@endif
 
{{-- ... --}}
```
It will display a message like this:

![image](https://user-images.githubusercontent.com/11309713/235356962-e8e0e2d9-942c-4042-a4b9-322bd487de4d.png)

This way, our Controller will prevent us from deleting the Category if there are any Questions related to it.

## Preventing deletion by hiding the Delete button
Another way to prevent deletion is to hide the delete button from the view. It is by adding a simple check in our View file:

First, let's load the count in our Controller:

*app/Http/Admin/CategoryController.php*
```
public function index()
{
    $categories = Category::withCount('questions')->get();
 
    return view('admin.categories.index', compact('categories'));
}
```
*resources/views/admin/categories/index.blade.php*
```
{{-- ... --}}
 
@if($category->questions()->count() === 0)
    <form action="{{ route('admin.categories.destroy', $category->id) }}"
          method="POST" onsubmit="return confirm('{{ trans('global.areYouSure') }}');"
          style="display: inline-block;">
        <input type="hidden" name="_method" value="DELETE">
        <input type="hidden" name="_token" value="{{ csrf_token() }}">
        <input type="submit" class="btn btn-xs btn-danger"
               value="Delete">
    </form>
@endif
 
{{-- ... --}}
```
And now our Delete is missing if there are any Questions related to it:

![image](https://user-images.githubusercontent.com/11309713/235357001-c3a9ff91-a6b0-46e3-8167-234dc919a512.png)

Our recommendation: for the best result - use both methods, protecting both front-end and back-end.

## More Mistakes?
So, these are only 3 typical mistakes with foreign keys, would you add more in the comments?

You can find more information about DB structure in the 2-hour video course How to structure databases in Laravel.
