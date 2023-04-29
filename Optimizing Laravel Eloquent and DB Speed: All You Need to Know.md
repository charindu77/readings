# Optimizing Laravel Eloquent and DB Speed: All You Need to Know
When it comes to the performance of the Laravel application, by far the no.1 problem I've seen is the database. DB Structure, Eloquent/SQL queries, and configuration - they all may cause many issues, so in this article, I've tried to compile "the ultimate guide" of what you need to know.

What we will talk about:

- Database Structure
- Indexing Columns
- Most Typical Eloquent Mistakes
- Eloquent vs Query Builder vs RAW Queries
- Caching
- MySQL Config
- Other DB Engines or NoSQL
So, prepare for a long read, with a lot of related links. Let's go!

## First Things First: Database Structure
Even before looking at Laravel, or even PHP code at all, we need to take a look at how our database is structured. Laravel is just a layer that would execute SQL queries on the structure that we had created and which would be hard to change.

So, when planning your database, you need to ensure that the structure itself is optimized. The problem is that there's no single way to build "the best" structure. Of course, it would be good to follow the DB normalization forms, but the real scenario depends on what operations you would have.

Here are just a few potential questions to consider:

- What tables/columns would be the most queried? (candidate for separating data into "main" and "secondary" tables)
- Will you filter by X column or just store data there? (candidate for JSON columns)
- How likely it is you would need to add X similar tables/columns in the future? (candidate for polymorphic relations)
- Will you have more insert or select operations? (candidate for NoSQL DB)
And you know what most developers don't do?

They don't talk enough to the BUSINESS guys who are responsible for the business logic of the app. Project owners should predict what may happen in the future, and their words should lead the technical DB structure decisions. Not the other way around.

I have a separate full 2-hour course called How to Structure Databases in Laravel, but let me give you one example.

**THE QUESTION: BelongsTo or BelongsToMany?**

This is one of those "million-dollar" questions that is not asked often enough. You should ask the client/manager or whoever is responsible for the project business logic:

- Will the article have only one category or multiple categories?
- Are you sure?
- Really really sure?
- Cause if we need to add multiple categories later, it would cost $X,XXX to restructure
On the other hand, if there's a small probability of that, the questions may be:

- Will articles have multiple categories?
- Do you have an example of such a multi-category article now?
- If we build that functionality now, we will deliver the first version a bit later, is that ok?
Business guys love when you talk to them in terms of money: cost to deliver, extra hours billed, potential future investments, risk management - that's their bread and butter. And, as developers, we need to ask them the right questions in their language, then follow up with our own advice, and the result should be a combined team effort.

## Indexing Columns Wisely
The database structure is not only about columns and relations.

As a separate question, we need to discuss indexing. On the surface, it's simple: if you put an index on some column, then searching by that column becomes faster.

I have run a very simple local experiment:

- 100,000 records in the users
- How fast would be an SQL query order by last_name with and without the index?
The users table migration, divided the name into first_name and last_name:
```
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->string('first_name');
        $table->string('last_name');
        $table->string('email')->unique();
        $table->timestamp('email_verified_at')->nullable();
        $table->string('password');
        $table->rememberToken();
        $table->timestamps();
    });
}
```
Then, a Factory with Faker:
```
class UserFactory extends Factory
{
    public function definition()
    {
        return [
            'first_name' => fake()->firstName(),
            'last_name' => fake()->lastName(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
            'remember_token' => Str::random(10),
        ];
    }
}
```
And then a Seeder for 100,000 users:
```
class UserSeeder extends Seeder
{
    public function run()
    {
        User::factory(100000)->create();
    }
}
```
Now, here's the SQL query I run directly in the SQL client (I use TablePlus):
```
select * from users order by last_name limit 10
```
The results are returned in 65 ms.

Now, what happens if we add the index to the last_name field?
```
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('first_name');
    $table->string('last_name')->index();
    // ... other columns
```
And we just re-seed the data. How much would the same SQL query take?
```
select * from users order by last_name limit 10

1 ms
```
Yes, 1 millisecond instead of 65 milliseconds.

That's how big may be the speed improvement if you use indexing.

That said, there's a cost to that. As the database engine needs more space to store the indexed data, the insert operation becomes slower: it needs to store the data, and then re-index the rows.

That's why you shouldn't put the index on all possible table columns, only the ones that are actually used in ordering or filtering.

## Optimizing Eloquent Queries
Eloquent is a great ORM engine that comes with Laravel. While it's convenient to write its queries, you have to be careful with what SQL commands are actually being executed under the hood, and how much memory it consumes.

I have a separate course called Better Eloquent Performance, but the most typical things to care about are listed in my video Eloquent Performance: TOP 3 Mistakes Developers Make. Let me summarize it for you, grouped into TWO most common mistakes.

## Eloquent Mistake 1. Too Many Queries: N+1
I know this is the elephant in the room, all the developers (hopefully) know about it: you have to use Eager Loading whenever possible.

Example from the docs:
```
// Wrong way:
$books = Book::all();
 
foreach ($books as $book) {
    echo $book->author->name;
}
```
In this case above, every $book->author will be queried from the database individually. So, the more books, the more DB queries.

Instead, you should load all the authors upfront, to have only TWO queries: one for all the books, and one for all the authors:
```
// Wrong way:
$books = Book::all();
 
// Right way:
$books = Book::with('author')->get();
```
But you may also introduce the N+1 query problem unintentionally, by using an unoptimized external package, or a deeper structure like accessors/mutators.

So, always monitor the number of queries. For that, the most popular tool is the Laravel Debugbar package, which shows the queries (and a lot of other things) at the bottom of your website.

If you work with API and non-visual projects, you may monitor the queries with Laravel Telescope.

Also, since Laravel 8.43, you can prevent [N+1 queries automatically](https://www.youtube.com/watch?v=bLWYbyKcfYI), I encourage you to enable it locally on all your projects.

## Eloquent Mistake 2. Too Much Data Downloaded
Whenever you perform an Eloquent query, it downloads some data from the database into a Laravel collection. But what if you actually need only a small amount of that data?

It's a pretty typical mistake I've seen developers make. It causes the amount of used RAM to be higher than expected, which may cause your server just crashing with no memory left.

The worst part is that you wouldn't easily see the memory usage while debugging one specific query or even one page/request unless you look at the right side of the Laravel Debugbar. So be careful with what you're requesting from the database.

A typical example is using with('some_relation') instead of withCount('some_relation') when all you need is the number of records.
```
// Wrong way:
$authors = Author::with('books')->get();
 
foreach ($authors as $author) {
    echo $author->name . ' (' . $author->books()->count() . ' books)';
}
```
In the case above, all book data will be downloaded, although you want to show only how many books the author has released.
```
// Right way:
$authors = Author::withCount('books')->get();
 
foreach ($authors as $author) {
    echo $author->name . ' (' . $author->books_count . ' books)';
}
```
**Notice:*** by the way, that $author->books()->count() is tricky in itself, watch this video to understand why those () symbols are very important.

Another typical example is downloading all the columns with Model::all() or Model::where('some_conditions')->get() although you're using only a few columns.
```
// Imagine this DB table has 20 columns:
$books = Book::all();
 
// But you're showing only a few:
foreach ($books as $book) {
    echo $book->title . ' (' . $book->release_year . ')';
}
 
// Instead, select only what you need:
$books = Book::select('title', 'release_year')->get();
```
Finally, you shouldn't perform the filters in the Collections, make your database work instead. What do I mean by that?
```
// This will download all the books and only then filter them
$books = Book::all()->where('release_year', now()->year);
 
// This will perform the filter on the DB level and download only what we need
$books = Book::where('release_year', now()->year)->get();
```
There are many more things you may do to optimize Eloquent queries, but what I would recommend first and foremost is testing your application locally with A LOT of data. Seriously, seed thousands/millions of fake records and see how the app performs. You may find a lot of unpleasant surprises.

## Eloquent vs Query Builder vs RAW Queries
Again, Eloquent is a great tool in terms of convenience for a developer, but often it performs not the most optimized queries under the hood.

You may find a better performance if you build the queries yourself with another Laravel tool called Query Builder.

On the surface, it may look the same as Eloquent, but it is slightly different. With Eloquent, you will get the advantage of some automation, like accessors/mutators, casting, and other transformations. But if you just need the data, without any of that "magic", and you're fine with writing a bit more code, Query Builder may be a better way.

A typical scenario is using whereHas() in Eloquent. Did you know that it runs two queries instead of one? Well, to be precise, it's a sub-select but with a full table scan.

I have a separate video about it, but the main message is this:
```
$transactions = Transaction::whereHas('category', function($q) {
    $q->where('project_id', 1);
})->get();
```
The Eloquent statement above will produce this query:
```
select * from transactions where exists
    (select * from categories where
        transactions.category_id = categories.id
        and project_id = 1)
```
See that second select *? Do we really need to perform SELECT ALL from that related table? We should use JOIN from SQL instead, right? So yeah, whereHas() doesn't do that, unfortunately.

Instead, you could use a Query Builder, like this:
```
$transactions = Transaction::select('transactions.*')
    ->join('categories', 'transactions.category_id', '=', 'categories.id')
    ->where('categories.project_id', 1)
    ->get();
```
This is the SQL executed:
```
select transactions.* from transactions
    join categories on transactions.category_id = categories.id
    where categories.project_id = 1
```
So, just a plain SQL query, it's much more efficient.

And there are other cases, where Eloquent "auto-magic" features just take too much time and you should consider a Query Builder instead, with the syntax like DB::table('table_name')->all_operations(...).

Speaking of plain SQL queries, no one actually forces you to use Eloquent or Query Builder as wrappers on top of the database. You can do something like this:
```
$books = \DB::select('select * from books join authors on books.author_id = authors.id');
```
In terms of performance, it will be the fastest, as it's literally executing the native DB code, without additional layers on top. But then, you're sacrificing the developer experience of writing less code, and features like Collections, Accessors, and other Laravel-related structures.

So I would use raw queries as the last resort. Also, don't forget to protect from the [SQL injection](https://www.youtube.com/watch?v=99Yit7WitxY), then!

## Caching: Look Ma, No Queries!
If you have optimized queries but you still feel there are too many of them running, and also you see there are repeating queries that often run on rarely-changed data, time to think about caching.

The logic is simple: when you run a DB query for the first time, put its result in the cache with some name assigned, and next time takes it from the cache instead of running another DB query. Until, of course, the data has changed, and then the cache should be cleared, with the updated data stored in the cache for the future.

A simple example of cached homepage data could be this:
```
$books = cache()->remember('homepage-books', $secondsToCache, function() {
    return Book:where(...)->where(...)->get();
})
```
When later you want to remove the item from the cache, just run Cache::forget('homepage-books').

You can also watch my full video about it.

There are many configuration options for the cache: engines like Memcached/Redis/DynamoDB, using Tags, and more, read about them in the official docs. Those features have nothing to do with Eloquent or even the Database engine at all, they can power any data you want to store in the cache with Laravel.

The downside or the risk of using the cache is that often developers forget to clear the cache, and load the data from the cache instead of refreshing it from the database.

For example, don't forget to run php artisan cache:clear every time you deploy the new code to the server, because it's highly likely that something may have changed in the database.

## DB Engine Config
If we zoom out of just Laravel or PHP code, we are left with the plain DB structure and SQL queries. Those also can be optimized, on the database level.

There are quite a few settings you can tune in MySQL itself, to improve its performance. To be honest, I'm not an expert in this, so I will just give you a few links to the tutorials written by professionals:

- [awesome-mysql-performance: A curated list of awesome links](https://github.com/Releem/awesome-mysql-performance)
- [Optimizing my.cnf for MySQL performance](https://www.red-gate.com/simple-talk/databases/mysql/optimizing-my-cnf-for-mysql-performance/)
- [MySQL Server Performance Tuning with Tips for Effective Optimization](https://www.devart.com/dbforge/mysql/studio/mysql-performance-tips.html)
- [MySQL Performance Tuning and Optimization Tips](https://phoenixnap.com/kb/improve-mysql-performance-tuning-optimization)
- [SQL Indexing and Tuning e-Book](https://use-the-index-luke.com/)
Also, there are tools like MySQLTuner: a script written in Perl that allows you to review a MySQL installation quickly and make adjustments to increase performance and stability.

Other DB Engines or NoSQL?
Of course, there are other DB engines, besides the most popular MySQL. Out of the box, Laravel has drivers for these:

- MySQL
- MariaDB
- PostgreSQL
- SQLite
- SQL Server
So maybe you have more experience in PostgreSQL and may choose that DB engine with its performance optimizations. Personally, I haven't worked with Postgre a lot but heard good things about its performance.

Also, if you have a lot of data, maybe you can turn to NoSQL engines, like MongoDB? Again, I don't have a lot of practical experience, and Laravel doesn't support that by default, so you need to use external packages like the most popular jenssegers/laravel-mongodb. There's also a tutorial about Laravel on the official MongoDB website.

Finally, a new growing tool I see mentioned a lot in 2022 is called [SingleStore](https://usefathom.com/blog/worlds-fastest-analytics). Back in 2021, Jack Ellis has written on the Fathom Analytics blog about how they moved from MySQL to Singlestore, and recently he released a full course on SingleStore for Laravel.

## Conclusion/Recap: "What to do" Checklist
So let's recap and simplify: what can you do if you see that your app performance is slow, and you think that database queries are the reason?

Here's my to-do list for you:

1. Install Laravel Debugbar / Telescope and browse around to check the number of queries on the pages/requests and RAM consumed.
2. Enable N+1 query restriction in AppServiceProvider.
3. Identify the slowest queries and optimize them.
4. Introduce caching where it makes sense.
5. Improve DB Structure if you see a problem there.
6. Configure/tune the DB engine
7. Potentially, change DB engine (but this should be the last resort cause it's a huge task)
I hope this article will give you some thoughts on how to optimize your Eloquent and DB performance. Wish you the best speed!
