# Eloquent Performance: 3 Most Common Mistakes
The performance of our applications is one of the top things we should care about. Inefficient Eloquent or DB queries are probably no.1 reason for bad performance. In this tutorial, I will show you the top 3 mistakes developers make when it comes to Eloquent performance, and how to fix them.

## Mistake 1: Too Many DB Queries
One of the biggest and most repeating mistakes is the N+1 query issue. This is generally caused by a lot of queries to the database and not using eager loading.

### Example 1: Eager Load Relationship
One of the most common example looks like this:

*app/Http/Controllers/PostController.php*
```
public function index()
{
    $posts = Post::all();
 
    return view('posts.index', compact('posts'));
}
```
And imagine you're using the Spatie Media Library package to load media files.

In the Blade file, you would use user and media relationships directly without preloading them:

*resources/views/posts/index.blade.php*
```
<ul>
    @foreach($posts as $post)
        <li>
            {{ $post->title }} / By {{ $post->user->name }}
            @foreach($post->getMedia() as $media)
                {{ $media->getUrl() }}
            @endforeach
        </li>
    @endforeach
</ul>
```
This produces a result similar to this, which contains a lot of database calls to get related users and media for a post:
![image](https://user-images.githubusercontent.com/11309713/235356139-745ea776-d195-447a-a712-7f4a60fc367e.png)

![image](https://user-images.githubusercontent.com/11309713/235356147-76fe929b-98d9-489b-ac30-18e3a7dc26d6.png)

To fix this, we can simply modify the controller to eager load the relationship like so:

1. Change all() to get()
2. Add the with(['user', 'media']) method to load the relationships
*app/Http/Controllers/PostController.php*
```
public function index()
{
    $posts = Post::with(['user', 'media'])->get();
 
    return view('posts.index', compact('posts'));
}
```
As a result, you will see only 3 queries being executed to load all the required data for the view:
![image](https://user-images.githubusercontent.com/11309713/235356177-23e07dd9-d656-4de6-a7e9-390766202083.png)

![image](https://user-images.githubusercontent.com/11309713/235356186-a2984487-6956-4b75-aacd-eccebcdcad16.png)

### Example 2: Counting Related Models
Another common mistake can be demonstrated by displaying how many posts each user has. See the example:

*app/Http/Controllers/UserController.php*
```
public function index()
{
    $users = User::with(['posts'])->get();
 
    return view('users.index', compact('users'));
}
```
And for the view, we can have one of the following: resources/views/users/index.blade.php
```
@foreach($users as $user)
    <li>{{ $user->name }} / Posts {{ $user->posts()->count() }}</li>
@endforeach
```
or
```
@foreach($users as $user)
    <li>{{ $user->name }} / Posts {{ $user->posts->count() }}</li>
@endforeach
```
Which one is correct? posts()->count() or posts->count()?

There's a big difference between them. Let's take for example
```
<li>{{ $user->name }} / Posts {{ $user->posts()->count() }}</li>
```
This will take the user and then attempt to load posts directly from the database due to us having posts() as it creates a new SQL query to get the count of posts for each user:
![image](https://user-images.githubusercontent.com/11309713/235356256-6c777fa7-53f7-4ce8-91f1-0579262d43c7.png)

![image](https://user-images.githubusercontent.com/11309713/235356264-9fbe0135-c9ca-4685-9fe3-04be70d8e662.png)

Instead, we should aim to have $user->posts as this will return our already preloaded posts:
```
<li>{{ $user->name }} / Posts {{ $user->posts->count() }}</li>
```
And this will use our already loaded data to reduce the number of queries we have:
![image](https://user-images.githubusercontent.com/11309713/235356278-7aeaa50a-7b40-46f2-b5ef-7b559a2d9341.png)

![image](https://user-images.githubusercontent.com/11309713/235356289-4ee25bc3-587e-4efa-8f57-44515927d36b.png)

## Mistake 2: Loading Too Much Data
Another common performance mistake is loading too much data while all you need is a small set of it.

### Example 1: with() VS withCount()
The first example will include a counting mistake which usually loads all the data to get a number out of it.

*app/Http/Controllers/UserController.php*
```
public function index()
{
    $users = User::with(['posts'])->get();
 
    return view('users.index', compact('users'));
}
```
*resources/views/users/index.blade.php*
```
<ul>
    @foreach($users as $user)
        <li>{{ $user->name }} / Posts {{ $user->posts->count() }}</li>
    @endforeach
</ul>
```
As you can see, we take all posts and then just count them in our view. This works, but produces more queries that it needs.
![image](https://user-images.githubusercontent.com/11309713/235356336-bbdcd50a-c250-4983-a7b9-d49ac618098f.png)

![image](https://user-images.githubusercontent.com/11309713/235356346-af424883-5fd9-4bdd-98f9-4ec11cda2fc5.png)

Let us update our code:

*app/Http/Controllers/UserController.php*
```
public function index()
{
    $users = User::withCount(['posts'])->get();
 
    return view('users.index', compact('users'));
}
```
*resources/views/users/index.blade.php*
```
<ul>
    @foreach($users as $user)
        <li>{{ $user->name }} / Posts {{ $user->posts_count }}</li>
    @endforeach
</ul>
```
By using withCount(['posts']) we are telling Eloquent to count the posts directly within the database. This produces only 1 query that is much more efficient than the previous ones:
![image](https://user-images.githubusercontent.com/11309713/235356388-9cb64c3f-32ab-40e6-81e9-aa5ce70a463b.png)

![image](https://user-images.githubusercontent.com/11309713/235356399-15b818c6-f003-4c01-a8ed-54d333ba2bcd.png)


Keep in mind that this will only retrieve counted results and is meant to optimize the counting of them. You will not be able to access any post information this way.

### Example 2: Loading Too Many Columns
Another example is loading only the required columns and not everything for a Model. This is great when you have big database tables, and you just need one or two columns.

In this case, we want to display the title of the post and the user's name, so we filter that in the Controller:

*app/Http/Controllers/PostController.php*
```
public function index()
{
    $posts = Post::with(
        [
            'user' => function ($query) {
                $query->select('id', 'name');
            },
            'media'
        ]
    )->get(['author_id', 'title']);
 
    return view('posts.index', compact('posts'));
}
```
And with our view file looks like this:

*resources/views/posts/index.blade.php*
```
<ul>
    @foreach($posts as $post)
        <li>
            {{ $post->title }} / By {{ $post->user->name }}
            @foreach($post->getMedia() as $media)
                {{ $media->getUrl() }}
            @endforeach
        </li>
    @endforeach
</ul>
```
We can see that we are only taking specific columns with our SQL query:
![image](https://user-images.githubusercontent.com/11309713/235356448-af68293a-0495-4b80-ac6d-fe82d25d536f.png)

![image](https://user-images.githubusercontent.com/11309713/235356460-60fdb209-a153-4f9c-8774-b963d371d5e0.png)

This way we are not loading any unnecessary data for posts and users.

## Mistake 3: Load Data First, Filter Later
This was noticed a lot in forums like laracasts where people tend to first do the database operation (like get()) and then do the filtering on the collection.

This is not the best way to do it as it will load all the data and then filter it out. To fix this, we can simply load the correct data from the database directly:

An example of a bad case. Focus on the get()->where(...) part, which first executes the database query and then applies filters:

*app/Http/Controllers/PostController.php*
```
public function index()
{
    $posts = Post::with(['user', 'media'])
        ->get()
        ->where('created_at', '>=', now()->subDays(7));
 
    return view('posts.index', compact('posts'));
}
```
This results in us getting all the data from the database and then filtering it with PHP which is slower:

![image](https://user-images.githubusercontent.com/11309713/235356485-3d1ad680-afcf-4686-92ca-9ef621a29b3b.png)

Imagine if we have 200,000 posts in our database! We don't need to load all posts even from 5 years ago, when all we need is 7 days of posts.

To get around this, we can change the order of the logic:

*app/Http/Controllers/PostsController.php*
```
public function index()
{
    $posts = Post::with(['user', 'media'])
        ->where('created_at', '>=', now()->subDays(7))
        ->get();
 
    return view('posts.index', compact('posts'));
}
```
And this will produce a correct database query to get only 7 days worth of posts:

![image](https://user-images.githubusercontent.com/11309713/235356506-910a18bb-9905-4bf8-aedc-4708dafb1daa.png)

## Final thoughts
As these are the most common patterns I've seen in my career, I hope this will help you to improve your code and make it more efficient.

To learn more about Eloquent and its performance you can check my courses like Eloquent: The Expert Level or Better Eloquent Performance.
