# 20+ Best Laravel Eloquent Packages
Eloquent is a great feature of Laravel, but also great is the list of packages that add more features on top of the framework. Let's explore them, in this article!

The title says 20 packages, but there are quite a few alternatives mentioned along the way, so the actual number is even bigger than this. Ready? Let's jump in!

## 1. spatie/laravel-query-builder
GitHub: https://github.com/spatie/laravel-query-builder

This package allows you to filter, sort and include eloquent relations based on a request.

Your users may type something like:

- /users?filter[name]=John
- users?include=posts
- users?sort=id
Then, your Controller will filter the records based on those parameters.

Code example:
```
$users = QueryBuilder::for(User::class)
    ->allowedFilters('name')
    ->allowedIncludes('posts')
    ->allowedSorts('id')
    ->get();
```
It reminds me of GraphQL logic where the server defines the rules for possible queries, and the client has freedom to choose what and how exactly they want to get in the results. I have a full Laravel+GraphQL course, if you're interested.

Alternative similar package: https://github.com/Tucker-Eric/EloquentFilter

## 2. spatie/laravel-searchable
GitHub: https://github.com/spatie/laravel-searchable

This package makes it easy to get structured search from a variety of Eloquent models, like a global search in your whole project.

How does it work?

First, you prepare your Models, providing what fields identify each of them:
```
use Spatie\Searchable\Searchable;
use Spatie\Searchable\SearchResult;
 
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
Similarly, you define other search result parameters for User model and other models.

Then, in your Controller, you may have this:
```
$searchResults = (new Search())
   ->registerModel(User::class, 'name')
   ->registerModel(Post::class, 'title')
   ->search('john');
```
The results will contain a multi-dimensional Eloquent collection, which you can iterate by various methods of the package. Here's a Blade example from the docs:
```
@foreach($searchResults->groupByType() as $type => $modelSearchResults)
   <h2>{{ $type }}</h2>
 
   @foreach($modelSearchResults as $searchResult)
       <ul>
            <li><a href="{{ $searchResult->url }}">{{ $searchResult->title }}</a></li>
       </ul>
   @endforeach
@endforeach
```
Alternative similar package: https://github.com/protonemedia/laravel-cross-eloquent-search

## 3. staudenmeir/eloquent-has-many-deep
GitHub: https://github.com/staudenmeir/eloquent-has-many-deep

This extended version of HasManyThrough allows relationships with unlimited intermediate models. It supports many-to-many and polymorphic relationships and all their possible combinations.

Consider this example from the Laravel documentation with an additional level: Country → has many → User → has many → Post → has many → Comment
```
class Country extends Model
{
    use \Staudenmeir\EloquentHasManyDeep\HasRelationships;
 
    public function comments()
    {
        return $this->hasManyDeep(Comment::class, [User::class, Post::class]);
    }
}
```
Then, you can use the N-th level relationship like a simple hasManyThrough:
```
Country::with('comments')->get();
```
I also recommend other Eloquent-related packages by Jonas Staudenmeir:

- [belongs-to-through](https://github.com/staudenmeir/belongs-to-through)
- [laravel-adjacency-list](https://github.com/staudenmeir/laravel-adjacency-list)
- [eloquent-json-relations](https://github.com/staudenmeir/eloquent-json-relations)
- [eloquent-eager-limit](https://github.com/staudenmeir/eloquent-eager-limit)

## 4. calebporzio/parental
GitHub: https://github.com/calebporzio/parental

Parental is a Laravel package that brings STI (Single Table Inheritance) capabilities to Eloquent. It's a fancy name for a simple concept: Extending a model, but referencing the same table.

Simple usage:
```
// The "parent"
class User extends Model
{
    use HasChildren;
}
 
// The "child"
class Admin extends User
{
    use HasParent;
 
    public function impersonate($user) {
        //...
    }
}
```
Then, in the Controller:
```
use App\Models\Admin;
 
// Returns "Admin" model, but reference "users" table:
$admin = Admin::first();
 
// Can now access behavior exclusive to "Admin"s
$admin->impersonate($user);
```
Without Parental, calling Admin::first() would throw an error because Laravel would be looking for an admins table. Laravel generates expected table names, as well as foreign keys and pivot table names, using the model's class name. By adding the HasParent trait to the Admin model, Laravel will now reference the parent model's class name users.

## 5. michaeldyrynda/laravel-cascade-soft-deletes
GitHub: https://github.com/michaeldyrynda/laravel-cascade-soft-deletes

This package solves a problem: when you use Soft Deletes in a parent-child hasMany relationship, if you soft-delete a parent record, children records do NOT get automatically soft-deleted.

For example, if you have Post -> hasMany -> Comment, both Models use Soft Deletes, and you want to delete the Post with its comments automatically, you need to provide this in the Post model:
```
class Post extends Model
{
    use SoftDeletes, CascadeSoftDeletes;
 
    protected $cascadeDeletes = ['comments'];
 
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}
```
Now you can delete a Post record with $post->delete(), and any associated Comment records will be deleted.

## 6. spatie/laravel-model-status
GitHub: https://github.com/spatie/laravel-model-status

Assign statuses to Eloquent models.

Imagine you want to have an Eloquent model hold a status. It's easily solved by just adding a status field to that model and be done with it, right?

But in case you need a history of status changes or need to store some extra info on why a status changed...

Just add a trait HasStatuses to your model:
``
use Spatie\ModelStatus\HasStatuses;
 
class YourEloquentModel extends Model
{
    use HasStatuses;
}
``
And then, in Controller, or wherever, you may perform these manipulations:
```
// set a status
$model->setStatus('pending', 'needs verification');
 
// set another status
$model->setStatus('accepted');
 
// specify a reason
$model->setStatus('rejected', 'My rejection reason');
 
// get the current status
$model->status(); // returns an instance of \Spatie\ModelStatus\Status
 
// get the previous status
$latestPendingStatus = $model->latestStatus('pending');

$latestPendingStatus->reason; // returns 'needs verification'
```

## 7. spatie/laravel-model-flags
GitHub: https://github.com/spatie/laravel-model-flags

Similar to the previous one about statuses, you may flag certain models with some condition, without having to add an additional column using migrations.

Code examples:
```
$user->hasFlag('receivedMail'); // returns false
 
$user->flag('receivedMail'); // flag the user as having received the mail
 
$user->hasFlag('receivedMail'); // returns true
It also provides scopes to quickly query all models with a certain flag.
 
User::flagged('myFlag')->get(); // returns all models with the given flag
User::notFlagged('myFlag')->get(); // returns all models without the given flag
```

## 8. owen-it/laravel-auditing
GitHub: https://github.com/owen-it/laravel-auditing

Laravel Auditing allows you to keep a history of model changes by simply using a trait. Retrieving the audited data is straightforward, making it possible to display it in various ways.

Just add a trait to your model:
```
class Article extends Model implements Auditable
{
    use \OwenIt\Auditing\Auditable;
 
    // ...
}
```
And then, you can retrieve the records:
```
// Get first available Article
$article = Article::first();
 
// Get all associated Audits
$all = $article->audits;
 
// Get first Audit
$first = $article->audits()->first();
 
// Get last Audit
$last = $article->audits()->latest()->first();
 
// Get Audit by id
$audit = $article->audits()->find(4);
```
There are alternative similar packages:

- [VentureCraft/revisionable](https://github.com/VentureCraft/revisionable)
- [WildsideUK/Laravel-Userstamps](https://github.com/WildsideUK/Laravel-Userstamps)

## 9. spatie/laravel-medialibrary
GitHub: https://github.com/spatie/laravel-medialibrary

Associate files with Eloquent models. This package can associate all sorts of files with Eloquent models, but mainly is used for saving images.

With this package, you can do something like:
```
$newsItem->addMedia($pathToFile)->toMediaCollection('images');
```
For that, prepare the Model with the interface and the trait:
```
use Spatie\MediaLibrary\HasMedia;
use Spatie\MediaLibrary\InteractsWithMedia;
 
class User extends Model implements HasMedia
{
    use InteractsWithMedia;
}
```
The information about the files is saved in the media DB table, using polymorphic relations.

You may also define the thumbnail image sizes (named "conversions") to be generated, whenever the image is saved, add this method in the same model:
```
use Spatie\MediaLibrary\MediaCollections\Models\Media;
 
public function registerMediaConversions(Media $media = null): void
{
    $this
        ->addMediaConversion('preview')
        ->fit(Manipulations::FIT_CROP, 300, 300)
        ->nonQueued();
}
```
Then, you can retrieve the media items of the model in various ways:
```
$mediaItems = $user->getMedia();
$url = $user->getFirstMediaUrl();
```

## 10. spatie/laravel-translatable
GitHub: https://github.com/spatie/laravel-translatable

Another package from Spatie, surprised? :)

This is a trait to make Eloquent models translatable. Translations are stored as json, so no extra table needed.

Code example:
```
$newsItem = new NewsItem; // This is an Eloquent model
$newsItem
   ->setTranslation('name', 'en', 'Name in English')
   ->setTranslation('name', 'nl', 'Naam in het Nederlands')
   ->save();
 
$newsItem->name; // Returns 'Name in English' given that the current app locale is 'en'
$newsItem->getTranslation('name', 'nl'); // returns 'Naam in het Nederlands'
 
app()->setLocale('nl');
 
$newsItem->name; // Returns 'Naam in het Nederlands'
```

## 11. Astrotomic/laravel-translatable
GitHub: https://github.com/Astrotomic/laravel-translatable

Alternative package to the Spatie's translatable package above (they even have the same name), but I listed this one separately because it has a different approach to the DB structure for translations. It does have a separate DB table for translations, instead of a JSON column.

But the code example is quite similar:
```
$post = Post::first();
echo $post->translate('en')->title; // My first post
 
App::setLocale('en');
echo $post->title; // My first post
 
App::setLocale('de');
echo $post->title; // Mein erster Post
```
Another example:
```
$data = [
  'author' => 'Gummibeer',
  'en' => ['title' => 'My first post'],
  'fr' => ['title' => 'Mon premier post'],
];
$post = Post::create($data);
 
echo $post->translate('fr')->title; // Mon premier post
```

## 12. spatie/laravel-tags
GitHub: https://github.com/spatie/laravel-tags

This package offers taggable behaviour for your models.

Just add a trait to the model:
```
use Spatie\Tags\HasTags;
 
class NewsItem extends Model
{
    use HasTags;
}
```
Then, you can perform manipulations like these:
```
// create a model with some tags
$newsItem = NewsItem::create([
   'name' => 'The Article Title',
   'tags' => ['first tag', 'second tag'], //tags will be created if they don't exist
]);
 
// attaching tags
$newsItem->attachTag('third tag');
$newsItem->attachTag('third tag','some_type');
$newsItem->attachTags(['fourth tag', 'fifth tag']);
 
// retrieving tags with a type
$newsItem->tagsWithType('categories');
$newsItem->tagsWithType('topics');
```
In addition to just tagging, this package has support for translating tags, multiple tag types and sorting capabilities.

Alternative similar package: https://github.com/rtconner/laravel-tagging

## 13. spatie/laravel-sluggable
GitHub: https://github.com/spatie/laravel-sluggable

This package provides a trait that will generate a unique slug when saving any Eloquent model.

Add a trait to your model, but you also need to implement the logic of the slugs yourself:
```
use Spatie\Sluggable\HasSlug;
use Spatie\Sluggable\SlugOptions;
 
class YourEloquentModel extends Model
{
    use HasSlug;
 
    public function getSlugOptions() : SlugOptions
    {
        return SlugOptions::create()
            ->generateSlugsFrom('name')
            ->saveSlugsTo('slug'); // you need to create this column in DB
    }
}
```
Then, in your Controller, you may do something like this:
```
$post = new Post();
$post->name = 'activerecord is awesome';
$post->save();
 
echo $post->slug; // outputs "activerecord-is-awesome"
```
Someone may ask, why you would need a package when there's a default Laravel Str::slug() method?

The thing is that this package takes care of unique slugs as well, and has more options to configure.

Alternative similar package: https://github.com/cviebrock/eloquent-sluggable

## 14. cybercog/laravel-love
GitHub: https://github.com/cybercog/laravel-love

Laravel Love lets people express how they feel about the content, by making any model reactable.

In your model, add Reacterable to who can react to items:
```
class User extends Authenticatable implements ReacterableInterface
{
    use Reacterable;
}
```
In the model of what to react to, add these:
```
class Comment extends Model implements ReactableInterface
{
    use Reactable;
}
```
And then, in Controllers, or wherever, you can use these:
```
$reacter->reactTo($reactant, $reactionType);
$reacter->reactTo($reactant, $reactionType, 4.0);
$isNotReacted = $reacter->hasNotReactedTo($reactant);
```

## 15. maize-tech/laravel-markable
GitHub: https://github.com/maize-tech/laravel-markable

A similar package to the one above, just maybe easier to use. This package allows you to easily add the markable feature to your application, as for example likes, bookmarks, favorites and so on.

In your model, add these:
```
use Maize\Markable\Markable;
use Maize\Markable\Models\Like;
 
class Course extends Model
{
    use Markable;
 
    protected static $marks = [
        Like::class,
    ];
}
```
And then, use it like this:
```
$course = Course::firstOrFail();
$user = auth()->user();
 
Like::add($course, $user); // marks the course liked for the given user
 
Like::count($course); // returns the amount of like marks for the given course
```
By default, the package supports four "marks":

- Bookmarks
- Favorites
- Likes
- Reactions
You may also implement your own custom mark.

## 16. renoki-co/befriended
GitHub: https://github.com/renoki-co/befriended

The third similar package, but this one is more directed towards social network functionality. Eloquent Befriended brings social media-like features like following, blocking and filtering content based on following or blocked models.

In your Model, you may specify who is Followable/Following:
```
use Rennokki\Befriended\Traits\Follow;
use Rennokki\Befriended\Contracts\Following;
 
class User extends Model implements Following {
    use Follow;
    ...
}
```
And then, example code from some Controller:
```
$alice = User::where('name', 'Alice')->first();
$bob = User::where('name', 'Bob')->first();
$tim = User::where('name', 'Tim')->first();
 
$alice->follow($bob);
 
$alice->following()->count(); // 1
$bob->followers()->count(); // 1
 
User::followedBy($alice)->get(); // Just Bob shows up
User::unfollowedBy($alice)->get(); // Tim shows up
```
In addition to following, there's also functionality for Blocking and Liking.

## 17. spatie/laravel-schemaless-attributes
GitHub: https://github.com/spatie/laravel-schemaless-attributes

Add schemaless attributes to Eloquent models. Wouldn't it be cool if you could have a bit of the spirit of NoSQL available in Eloquent? This package does just that. It provides a trait that when applied on a model, allows you to store arbitrary values in a single JSON column.

To use, add a migration for all models where you want to add schemaless attributes to:
```
Schema::table('your_models', function (Blueprint $table) {
    $table->schemalessAttributes('extra_attributes');
});
```
Then, prepare the model:
```
use Spatie\SchemalessAttributes\Casts\SchemalessAttributes;
 
class TestModel extends Model
{
    public $casts = [
        'extra_attributes' => SchemalessAttributes::class,
    ];
 
    public function scopeWithExtraAttributes(): Builder
    {
        return $this->extra_attributes->modelScope();
    }
 
    // ...
}
```
And then, you can use it like this:
```
// add and retrieve an attribute
$yourModel->extra_attributes->name = 'value';
$yourModel->extra_attributes->name; // returns 'value'
 
// you can also use the array approach
$yourModel->extra_attributes['name'] = 'value';
$yourModel->extra_attributes['name'] // returns 'value'
 
// setting multiple values in one go
$yourModel->extra_attributes = [
   'rey' => ['side' => 'light'],
   'snoke' => ['side' => 'dark']
];
```

## 18. kirschbaum-development/eloquent-power-joins
GitHub: https://github.com/kirschbaum-development/eloquent-power-joins

This package allows to use a different syntax for DB joins, kind of a middle ground between Eloquent and Query Builder.

You can read a more detailed explanation on the problems this package solves on this blog post.

In your Model, define this:
```
use Kirschbaum\PowerJoins\PowerJoins;
 
class User extends Model
{
    use PowerJoins;
}
```
Then, you can use the joinRelationship() syntax:
```
User::joinRelationship('posts', function ($join) {
    $join->where('posts.approved', true);
})->toSql();
```

## 19. spatie/eloquent-sortable
GitHub: https://github.com/spatie/eloquent-sortable

This package provides a trait that adds sortable behaviour to an Eloquent model.

In your model, add this:
```
class MyModel extends Model implements Sortable
{
    use SortableTrait;
 
    public $sortable = [
        'order_column_name' => 'order_column',
        'sort_when_creating' => true,
    ];
}
```
You need to have order_column in your DB table, that field would be used for sorting.

Then, when creating new records, that field name would be auto-saved:
```
$myModel = new MyModel();
$myModel->save(); // order_column for this record will be set to 1
 
$myModel = new MyModel();
$myModel->save(); // order_column for this record will be set to 2
```
And then, you can get the ordered records:
```
$orderedRecords = MyModel::ordered()->get();
```
And you can easily change the order of a current object, with these methods:
```
$myModel->moveOrderDown();
$myModel->moveOrderUp();
$myModel->moveToStart();
$myModel->moveToEnd();
```
Alternative similar packages:

boxfrommars/rutorika-sortable
Kyslik/column-sortable

## 20. topclaudy/compoships
GitHub: https://github.com/topclaudy/compoships

Compoships offers the ability to specify relationships based on two (or more) columns in Laravel's Eloquent.

Code example:
```
namespace App;
 
use Illuminate\Database\Eloquent\Model;
 
class User extends Model
{
    use \Awobaz\Compoships\Compoships;
 
    public function tasks()
    {
        return $this->hasMany(Task::class,
            ['team_id', 'category_id'],
            ['team_id', 'category_id']);
 
        // Schematically:
        // return $this->hasMany('B',
        //    ['foreignKey1', 'foreignKey2'],
        //    ['localKey1', 'localKey2']);
 
    }
}
```
So that's it for this long list, but it may be even longer. Did we miss any package? Add them in the comments!
