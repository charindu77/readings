# Laravel Multiple Model Search: Queries, Scout, Packages
If you want to search in multiple Eloquent models - like posts, videos, and courses - there are a lot of different ways, with or without external packages and tools. Let's explore them in this article.

First, the example we will be working on. Three simple DB tables:

- posts
- videos
- courses
- 
![image](https://user-images.githubusercontent.com/11309713/235345001-36c73b62-5977-45ae-9d8b-7ea7e49795e7.png)

![image](https://user-images.githubusercontent.com/11309713/235345012-b98bf213-4c80-494d-ac26-f5b2e843f39d.png)

![image](https://user-images.githubusercontent.com/11309713/235345030-f9a9c842-f8ee-4877-b532-8f18d13d490a.png)

Each table has its Eloquent model, and we want to query those tables and get results like this:

![image](https://user-images.githubusercontent.com/11309713/235345049-18b42a01-d104-4a4c-936d-7a06c8b6758b.png)

So, what are our options? Let's go from the most simple one to adding more tools later.

## Simple: Three Separate Queries
This is the most straightforward, almost "dumb" way to perform the task. Why group or optimize something, if you can just query the tables separately?

Controller:
```
$keyword = request('keyword');
$results['posts'] = Post::where('title', 'like', '%' . $keyword . '%')->get();
$results['videos'] = Video::where('title', 'like', '%' . $keyword . '%')->get();
$results['courses'] = Course::where('title', 'like', '%' . $keyword . '%')->get();

return view('search', compact('results'));
```
And then in the Blade, you just show three separate almost identical blocks, with some Tailwind (or whatever you prefer) styling:
```
<h2 class="text-2xl mb-4">Search results</h2>
 
<div class="font-bold mt-2 mb-2">Posts:</div>
@if ($results['posts']->count())
    <ul class="list-inside">
        @foreach ($results['posts'] as $post)
            <li class="list-disc">{{ $post->title }}</li>
        @endforeach
    </ul>
@else
    No results.
@endif
 
<div class="font-bold mt-2 mb-2">Videos:</div>
@if ($results['videos']->count())
    <ul class="list-inside">
        @foreach ($results['videos'] as $video)
            <li class="list-disc">{{ $video->title }}</li>
        @endforeach
    </ul>
@else
    No results.
@endif
 
<div class="font-bold mt-2 mb-2">Courses:</div>
@if ($results['courses']->count())
    <ul class="list-inside">
        @foreach ($results['courses'] as $course)
            <li class="list-disc">{{ $course->title }}</li>
        @endforeach
    </ul>
@else
    No results.
@endif
```
So there you go, the result is here, nothing more to write in this tutorial, right?

Well... I will disappoint you: there are other, more elegant ways.

## Laravel Scout with Database Driver
If you want to avoid long "like" statements in the Controller above, especially if you want to perform the search in a few columns of each model, Laravel Scout may help.
```
// Without Scout:
Post::where('title', 'like', '%' . $keyword . '%')->get();
 
// With Scout:
Post::search($keyword)->get();
```
In other words, we're offloading the search logic to a separate search engine, instead of Controller.

Historically, Laravel Scout has been an engine to perform a text-based search with external systems, like Algolia or Elasticsearch.

But since Laravel 9, you can use a database driver.

I have a full YouTube [video](https://www.youtube.com/watch?v=7KbJIBM5VHg) about it, but we will quickly implement it in our example.

First, we install Scout, as per instructions:
```
composer require laravel/scout
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```
Then, in the config/scout.php file we need to specify that we use the "database" driver:

#### config/scout.php
```
return [
 
    /*
    |--------------------------------------------------------------------------
    | Default Search Engine
    |--------------------------------------------------------------------------
    |
    | This option controls the default search connection that gets used while
    | using Laravel Scout. This connection is used when syncing all models
    | to the search service. You should adjust this based on your needs.
    |
    | Supported: "algolia", "meilisearch", "database", "collection", "null"
    |
    */
 
    'driver' => env('SCOUT_DRIVER', 'database'),
```
Or, of course, you can specify it in your .env file as SCOUT_DRIVER=database.

Then, we add the Searchable logic into all three of our Eloquent models.

#### app/Models/Post.php:
```
use Laravel\Scout\Searchable;
 
class Post extends Model
{
    use Searchable;
 
    public function toSearchableArray()
    {
        return [
            'title' => $this->title,
        ];
    }
}
```
#### app/Models/Video.php:
```
use Laravel\Scout\Searchable;
 
class Video extends Model
{
    use Searchable;
 
    public function toSearchableArray()
    {
        return [
            'title' => $this->title,
        ];
    }
}
```
#### app/Models/Course.php:
```
use Laravel\Scout\Searchable;
 
class Course extends Model
{
    use Searchable;
 
    public function toSearchableArray()
    {
        return [
            'title' => $this->title,
        ];
    }
}
```
In all those three models, we specify to search only in the title column, which is names coincidentally the same. In your real case, the search criteria and field names may be different.

And then, in the Controller, we can do this:
```
$keyword = request('keyword');
 
$results['posts'] = Post::search($keyword)->get();
$results['videos'] = Video::search($keyword)->get();
$results['courses'] = Course::search($keyword)->get();
 
return view('search', compact('results'));
```
All the searches will return the full Eloquent models by default, so the Blade part doesn't change - we're doing the @foreach over each of those three parts.

#### Laravel Scout with External Driver?
I've already mentioned a few Scout drivers above. The question is: why you would use them, instead of the default Database driver?

Systems like Algolia or Meilisearch are specifically built for text-based search. It means they will store the search index and perform the search more optimally.

Not only that, they have additional functions like "fuzzy search" (you love when Google fixes your typos, right?) and various settings on how to sort/filter the results, without changing your code.

In this article, I will not give you examples of those, because the code of our Controller and Blade View wouldn't change, you would only change the search engine that operates.

Out of the box, Laravel supports Algolia and Meilisearch. There is a fundamental difference between those:

- Algolia is a service: free only with a limited number of records/searches, but very easy to install and configure (by the way, Laravel Daily uses Algolia)
- Meilisearch is a free open-source software that you need to install and configure yourself. But they also have a cloud-based premium version, like Algolia.
There's also Elasticsearch that used to be very popular so you may find a lot of Laravel tutorials about it, but not as many since Meilisearch took over its popularity, in recent years. Mostly, because Elasticsearch doesn't have the default Laravel Scout driver, you need to use a package like this one.

Here's the screenshot of the current search index of Laravel Daily, from the Algolia dashboard.

![image](https://user-images.githubusercontent.com/11309713/235345249-c5391473-a722-4f02-8d7e-aa2486b42630.png)

Read more about all the drivers and their functionality in the official Laravel Scout documentation.

## Spatie Laravel Searchable: Grouped Results
What if we wanted to perform one sentence in the Controller instead of three? Also, what if we wanted to have one @foreach loop in the results instead of three separate ones?

Well, the Spatie package laravel-searchable to the rescue. Let's see what it can do for us.

We install it:
```
composer require spatie/laravel-searchable
```
Then, our Models should implement the Searchable interface and add the getSearchResult() method, to specify which fields would exist in the result:

## app/Models/Post.php:
```
use Spatie\Searchable\Searchable;
use Spatie\Searchable\SearchResult;
 
class Post extends Model implements Searchable
{
 
    public function getSearchResult(): SearchResult
    {
        return new \Spatie\Searchable\SearchResult(
            $this,
            $this->title,
        );
    }
}
```
#### app/Models/Video.php:
```
use Spatie\Searchable\Searchable;
use Spatie\Searchable\SearchResult;
 
class Video extends Model implements Searchable
{
 
    public function getSearchResult(): SearchResult
    {
        return new \Spatie\Searchable\SearchResult(
            $this,
            $this->title,
        );
    }
}
```
#### app/Models/Course.php:
```
use Spatie\Searchable\Searchable;
use Spatie\Searchable\SearchResult;
 
class Course extends Model implements Searchable
{
 
    public function getSearchResult(): SearchResult
    {
        return new \Spatie\Searchable\SearchResult(
            $this,
            $this->title,
        );
    }
}
```
Again, these models look almost identical in our case, but you may also specify the URL for the click from the search results page:
```
class Post extends Model implements Searchable
{
     public function getSearchResult(): SearchResult
     {
        $url = route('posts.show', $this->slug);
 
         return new \Spatie\Searchable\SearchResult(
            $this,
            $this->title,
            $url
         );
     }
}
```
Then, in the Controller, we have just this:

#### use Spatie\Searchable\Search;
 ```
// ...
 
$results = (new Search())
    ->registerModel(Post::class, 'title')
    ->registerModel(Video::class, 'title')
    ->registerModel(Course::class, 'title')
    ->search(request('keyword'));
 
return view('search', compact('results'));
```
And then, the most important thing - the results are automatically grouped for us!
```
@forelse($results->groupByType() as $type => $modelSearchResults)
    <div class="font-bold mt-2 mb-2">{{ ucfirst($type) }}</div>
    <ul class="list-inside">
        @foreach($modelSearchResults as $searchResult)
            <li class="list-disc">
                {{ $searchResult->title }}
            </li>
        @endforeach
    </ul>
@empty
    No results.
@endforelse
```
So, there's only ONE @foreach instead of three separate ones.

Now, the downside of this is that if you don't have results for any of those three models, then they will not be part of $results at all. But maybe that's exactly the behavior you want. Your personal choice is whether to use this package.

## Alternative No-Grouping Package: Cross Eloquent Search
Another well-known Laravel package to search in multiple Eloquent models is protonemedia/laravel-cross-eloquent-search.

The logic is pretty simple to the Spatie package above, but the results are NOT grouped, so you will have one @foreach of all results in Blade, but will have to show the prefix of class_basename() to show which model it is.

We install the package:
```
composer require protonemedia/laravel-cross-eloquent-search
```
Then, we may NOT need to configure anything in the Models!

And our Controller looks like this:
```
use ProtoneMedia\LaravelCrossEloquentSearch\Search;
 
$results = (new Search())
    ->registerModel(Post::class, 'title')
    ->registerModel(Video::class, 'title')
    ->registerModel(Course::class, 'title')
```
Finally, in the Blade, we have something like this:
```
@foreach($results as $searchResult)
    <li class="list-disc">
        {{ class_basename($searchResult) }}:
        {{ $searchResult->title }}
    </li>
@endforeach
```
As you can see, class_basename($searchResult) will return just the PHP class, which is, in our case, exactly what we need: "Post", "Video", or "Course".

So, use this package if you want to have just one ungrouped list of all possible results.

## Conclusion: Global Search is Simple. Or Hard.
As you saw above, there are multiple ways to search in multiple models, depending on how you want to present your results.

With external drivers powered by Laravel Scout, this topic of a text-based search may go much deeper, with external functionality and performance optimizations. But these are topics for separate advanced tutorials or courses, let me know your questions in the comments, so I may follow up in the future!
