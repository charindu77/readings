# How to Name Things in Laravel
How should you name your Controllers: singular or plural? With other similar questions, naming things consistently is important for code readability, compliance with standards, and avoiding errors. In this tutorial, we will cover a dozen tips for naming different things in Laravel and PHP, including Models, Controllers, Blade files, Migrations, and more.

Keep in mind that most of those are not strict rules, you can name things however you want, as long as it works. However, the problems with non-standard naming can be these:

You may need to write additional code to configure things to work properly
New developers on your team may need more time to understand the code
If things are named inconsistently, other developers need more brainpower/time to decide how to name things in their code
In some cases, it may even lead to bugs
So, I do encourage you to keep your code as close as possible to the standard naming conventions. Here are the examples.

## How to Name Models in Laravel?
The rule of thumb in Laravel is to use a singular form for model names, like Book instead of Books.

This is because when you generate a model using php artisan make:model, Laravel assumes that the corresponding database table will be plural.

Using English words for naming is also encouraged to make the code readable for others who don't speak your native language. Also, you may break the Laravel stuff. For example, in German, for model Buch Laravel would assume that the table should be Buchs, with s at the end. Because Laravel will try to make plural. But in German plural is Bücher. And imagine for a non-German speaker to read that class, it won't make sense what it does and what it is.

If you need to name a Model and a DB table with multiple words, use PascalCase (e.g. BookReview), and Laravel will assume the table name is in a plural form (e.g. book_reviews). However, if you want to customize the table name, you can do so in the protected $table property of the model.
```
class BookReview extends Model
{
    protected $table = 'books_reviews'; 
}
```
When writing Model Relations use camelCase naming, for example publishedLessons() and not published_lessons():
```
use Illuminate\Database\Eloquent\Relations\HasMany;
 
class Course extends Model
{
    public function publishedLessons(): HasMany
    {
        return $this->hasMany(Lesson::class)->published();
    }
}
```
## How to Name Controllers in Laravel
While the name of the controller can be singular or plural (e.g. BookController or BooksController), it is recommended to use the singular form.

When you generate a model using php artisan make:model Book -c, Laravel automatically generates a singular form for the controller (e.g. BookController). There's no strict rule here, it's just what Laravel generates.

## How to Name Blade Files in Laravel
For Blade files, it's essential to keep files organized in subfolders based on the object (Model) they work with and their action (Controller method).

- Inconsistent: BookView.blade.php for a show Controller method
- Consistent: books/index.blade.php, books/show.blade.php, etc.
This advice comes from the Laravel Resource Controller: the names of the methods there should be probably the names of Blade files. By following this convention, it's easier to find related files when needed, even in larger projects.

## How to Name Migrations in Laravel
Wait, migrations? I thought we shouldn't name the migration files manually, as it happens automatically?

But it's important how exactly you name the parameter in the make:migration command.

When creating Migration it creates a file xxxx_create_books_table.php, and that is not a coincidence. When you want to create a new table, the syntax for the command for example should be php artisan make:migration create_XXXXX_table. This way, Laravel will help you generate a migration file, where the first line will tell schema to create table with the exact name you provided.

Example for php artisan make:migration create_book_reviews_table:
```
return new class extends Migration
{
    public function up()
    {
        Schema::create('book_reviews', function (Blueprint $table) { 
            $table->id();
            $table->timestamps();
        });
    }
};
```
And if your Migration isn't about creating a new table, but about modifying existing ones, when calling the make:migration command instead of starting with create you can do for example php artisan make:migration modify_whatever_to_book_reviews_table.

This will instead of Schema::create() make Schema::table() and will fill the table name automatically.
```
return new class extends Migration
{
    public function up()
    {
        Schema::table('book_reviews', function (Blueprint $table) { 
            //
        });
    }
};
```
## How to Name Foreign Keys in Laravel
When creating migrations for foreign ID column, try to always use model_id name. For example, if post belongs to a user, column in Post table should be user_id, and not something like posted_by_user.

This way Laravel will automatically perform its magic when defining relation and will automatically add a constrain to users table ID column.
```
return new class extends Migration
{
    public function up()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->foreignId('user_id')->constrained(); 
        });
    }
};
```
## Suffixes or Endings for Class Names
Use class endings or suffixes to reflect the purpose of the class, such as Controller, Seeder, Factory, or Model (e.g. BookController, or BookSeeder).

Follow this convention to make it immediately clear what each class does, for any developer in the future.

## PSR: Consistency Across Team Members
To ensure consistency across team members and make the code easy to read and maintain, use PSR-12 standards for PHP. Some examples of PSR-12 code standards include using spaces instead of tabs, naming conventions for classes, and using snake_case for function names.

Laravel follows the PSR-2 coding standard. If you want to use the same standards in your code, the easiest way would be to use Laravel official package Laravel Pint.

By adhering to these standards, it's easier to read and maintain code, especially in larger projects that may have multiple team members working on the same codebase. Additionally, tools like PHPStorm can help automatically reformat code to comply with these standards.
