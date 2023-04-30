# UUID in Laravel: All You Need To Know
If you want to replace DB auto-increment IDs with something more sophisticated, one of the solutions is UUID. In this article, I will show you how UUID columns work in Laravel, and what are the options and tools to use them.

What is UUID and Why You Would Need It?
If we take a look at Wikipedia, this is the definition:

"A universally unique identifier (UUID) is a 128-bit label used for information in computer systems. The term globally unique identifier (GUID) is also used."

This term comes not from Laravel or PHP, it's a general concept for IT projects, to uniquely identify records.

Some UUID examples:
```
b898564a-4ce8-4114-9067-142b437075ae
236d75e7-b7e2-41f4-af8f-7b9e9cf32ed9
b7c91937-74b2-4230-97cf-92433cc6dd9a
```
You would save these identifiers in the database, instead of (or in addition to) a typical ID column.

So, if you have these URLs in your project:

- yourproject.com/posts/1
- yourproject.com/transactions/123/edit
With UUID, they would look like this:

- yourproject.com/posts/b898564a-4ce8-4114-9067-142b437075ae
- yourproject.com/transactions/236d75e7-b7e2-41f4-af8f-7b9e9cf32ed9/edit
Why would you do that? A few reasons. Two of them are more practical for any project, others are more theoretical for bigger systems.

- They cannot be guessed: from the security point of view, these kinda-encoded URLs don't give any chance to guess the ID of another record that may belong to another user.
- They hide the real numbers: if you have only 3 records in the database but don't want your visitors to know that your startup is small, UUIDs are to the rescue.
- Database insert is more flexible: typical insert with auto-increment IDs takes additional DB resources to generate that ID. If we switch that responsibility to the application, we potentially free DB resources.
- Database scalability: if we need to split the database into servers, or merge with another database if acquired by another startup, there won't be conflicts in IDs.
Ok, now let's get from why to the how.

In fact, there are two main ways to use UUIDs, and I already mentioned them above:

- Use UUID as a primary key and remove the auto-increment ID
- Use UUID as an additional visible key and still keep the auto-increment ID, just hidden from public
Let's take a look at each of the approaches.

## UUID as Primary Key
In Laravel, this feature became much easier since v9.30. If you're a visual learner, you can watch my video about it:
[VIDEO](https://youtu.be/1DGaphgk2kI)

These are the things you need to do:

In the migration file, have this:
```
// Instead of: $table->id();
$table->uuid('id')->primary();
```
Then, in the Model, have this trait:
```
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
 
class Article extends Model
{
    use HasUuids;
}
```
And that's it, you save the record with Eloquent and get UUID auto-generated for you:
```
$article = Article::create(['title' => 'Traveling to Europe']);
echo $article->id; // "8f8e8478-9035-4d23-b9a7-62f4d2612ce5"
```
If you work with older Laravel versions, you would need to generate the UUID manually. There are a few different ways to do it, but the most typical one involved a Trait anyway, just Laravel 9.30 introduced its own "official" Trait.

Previously, you would create something like this:

#### app/Traits/Uuid.php:
```
namespace App\Traits;
use Illuminate\Support\Str;
 
trait Uuid
{
    protected static function boot()
    {
        parent::boot();
 
        static::creating(function ($model) {
            $model->id = (string)Str::uuid();
        });
    }
 
    public function getIncrementing() : bool
    {
        return false;
    }
 
 
    public function getKeyType() : string
    {
        return 'string';
    }
}
```
As you can see, in addition to generating the UUID, we also override two properties of the Eloquent Model: disable the auto-increments and specify that the primary key is a string now, and not an integer anymore.

And then we add this Trait to our Model:
```
use App\Traits\Uuid;
 
class Post extends Model
{
    use HasFactory, Uuid;
 
    // ...
}
```
And then, the URL part would work well automatically even with Route Model Binding.

So if you do:
```
Route::resource('posts', PostController::class);
 
// PostController
public function show(Post $post) {
    // ...
}
```
It will automatically form the URL with the UUID field.

## What About Foreign Keys?
We replaced the auto-increment ID with our UUID, so what about other tables related to it?

For example, if we have a Post model with UUID as a primary key, what should be inside the Comment model with ->belongsTo(Post::class)?

In migrations, there's a specific field type:
```
$table->foreignUuid('post_id')->constrained();
```
It's identical to a foreignId() method, meaning it would create both the field and the foreign key.

You don't need to change anything else in the Eloquent model, and you can still use the same ->belongsTo() method.

Yes, it means that the same long string like "b898564a-4ce8-4114-9067-142b437075ae" will be also stored in the comments DB table, as well as the posts DB table. And this is one of the disadvantages of the UUIDs - you need more DB storage for them.

## UUID as Secondary Key
Another way (and I personally prefer this one) is to leave a typical ID as default, and add a UUID column as an additional field.

So, migration would look like this:
```
$table->id();
$table->uuid('uuid');
// ... all other fields
```
Then, the new Laravel 9.30 way doesn't apply because it works only for primary keys, so we need to put our own Trait, just filling in the UUID:

#### app/Traits/Uuid.php:
```
namespace App\Traits;
use Illuminate\Support\Str;
 
trait Uuid
{
    protected static function boot()
    {
        parent::boot();
 
        static::creating(function ($model) {
            $model->uuid = (string)Str::uuid();
        });
    }
}
```
As you can see, in this case, we don't override anything with increments and the key, we leave Eloquent defaults as they are.

And then add the Trait to the Model:
```
use App\Traits\Uuid;
 
class Post extends Model
{
    use HasFactory, Uuid;
 
    // ...
}
```
So, saving data is done, but the additional work with this approach is that we need to manually specify to "hide" the ID field and perform the lookup search by UUID instead.

A few options here.

If you use just the show() method with Route Model Binding, you can define the key directly in the Route:
```
Route::get('/posts/{post:uuid}', [PostController::class, 'show']);
```
If you use the full Resource Controller, then it's better to specify the UUID as the key in the Model itself.
```
use App\Traits\Uuid;
 
class Post extends Model
{
    use HasFactory, Uuid;
 
    // ...
 
    public function getRouteKeyName()
    {
        return 'uuid';
    }
}
```
Then, you can safely use Route::resource(), and all the show() / edit() / update() / destroy() method will lookup the model by UUID.

What is the benefit of this approach? Well, we have the best of both worlds:

- We still have auto-incremented ID for internal usage, so we would know how many records we have, we can order by them quickly when using SQL Client or in Eloquent queries.
- But we don't expose that ID to the public, so our app users work only with the non-guessable UUID.

## Are UUIDs Slower? Yes, But By How Much?
Some of you may think: if we store UUIDs in the database and perform queries by them, they should be quite slow, as they are strings and not integers, right? So they are slower to query, and even their indexing may take more resources.

And you're right, UUIDs are somewhat a hit to performance, in MySQL and other databases. There are various articles/videos about that:

- [UUIDs are Popular, but Bad for Performance — Let’s Discuss](https://www.percona.com/blog/2019/11/22/uuids-are-popular-but-bad-for-performance-lets-discuss/)
- [UUID performance in MySQL?](https://stackoverflow.com/questions/2365132/uuid-performance-in-mysql)
- [Video: UUIDs are Bad for Performance in MySQL - Is Postgres better? Let us Discuss](https://www.youtube.com/watch?v=Y5mWz4vK10A)
But, looking at those benchmarks, I was wondering: when would you reach the stage where this performance would actually be noticeable?

In my career, I've been working with smaller projects, where DB tables reached 100,000k records or even less, so for those numbers, you should not feel any performance difference for UUIDs.

Also, of course, you should index the UUID column:
```
$table->uuid('uuid')->index();
```
This will make the insert a bit slower, but the lookup will be much faster.

## UUID Versions and ULID
The last thing worth mentioning is that there are different versions or "generations" of UUIDs. You can read about them in this article, at the moment of writing this post there are eight different versions.

![image](https://user-images.githubusercontent.com/11309713/235344776-81f434c6-b215-4cd6-8ffe-18dfb0b509bf.png)

Their differences are about algorithms and how to ensure the uniqueness of those identifiers.

Do you actually need to know all those versions? To be honest, probably not. Currently, the Laravel implementation uses UUID v4 but will switch to UUID v7 from Laravel 10. And if you want to use specific packages for UUIDs, Ben Ramsey's package [supports UUID v8](https://laravel-news.com/ramsey-uuid-v8).

To make matters even more confusing, there are also ULIDs: Universally Unique Lexicographically Sortable Identifiers. You can read its specification here on Github, and if you do want to use them, the good news is that Laravel supports ULID out of the box, with the same Laravel 9.30 update, you just need to use a different Trait: use HasUlids; instead of HasUuids;.

A few more resources on ULIDs:

- [Going deep on UUIDs and ULIDs](https://www.honeybadger.io/blog/uuids-and-ulids/)
- [The Wild World of Unique Identifiers (UUID, ULID, etc)](https://medium.com/geekculture/the-wild-world-of-unique-identifiers-uuid-ulid-etc-17cfb2a38fce)
- [ULID vs UUID: Sortable Random ID Generators for JavaScript](https://blog.bitsrc.io/ulid-vs-uuid-sortable-random-id-generators-for-javascript-183400ef862c)
## Conclusion: Do You Need To Be Unique?
In my experience, UUIDs are great when you actually need them and when you have a clear reason to make your structure more complex and introduce universally unique identifiers.

For a smaller project, you don't really need to hide/protect your auto-incremented IDs that much, probably. Worst case, you may set the increment value in MySQL to some 12345, huh?
