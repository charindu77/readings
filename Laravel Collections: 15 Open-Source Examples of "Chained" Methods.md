# Laravel Collections: 15 Open-Source Examples of "Chained" Methods
Collections in Laravel are "hidden gems": not everyone is using them. They are especially effective when performing MULTIPLE operations with data - so-called "chains". I've gathered 15 real-life examples from open-source Laravel projects. The goal is not only to show the Collection methods but also the practical scenarios of WHEN to use them.

This long tutorial is a text-version of my video course with the same examples.

So, let's dive into examples, one by one, with the links to the original sources.

## Example 1: map + implode
Let's start this tutorial about Laravel Collection Chains with a very simple example, with a chain of two methods. The goal here is to show permissions divided by a new HTML tag.

Code:
```
$role = Role::with('permissions')->first();
$permissionsListToShow = $role->permissions
    ->map(fn($permission) => $permission->name)
    ->implode("<br>");
```
The initial value of $role->permissions is permission objects, and we are interested only in the name field.

![image](https://user-images.githubusercontent.com/11309713/235352259-e33a773d-5800-4aae-823d-53cd17f19c71.png)


Then we do map through those permissions, where we new collection containing only the names.

![image](https://user-images.githubusercontent.com/11309713/235352271-ab4209c1-f27d-419a-afda-f68f2c870c5b.png)


And finally after implode() we get:
```
"manage users<br>manage posts<br>manage comments"
```
The GitHub repository with code can be found [here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L26). Inspiration source: [Bottelet/DaybydayCRM](https://github.com/Bottelet/DaybydayCRM/blob/a5719a23bdc2e29e021e86b97a1116ed1fd683c2/app/Http/Controllers/RolesController.php).

## Example 2: map + max
You have an array of some kinds of scores, and then you have the collection of the scores from the database or from elsewhere.

Code:
```
$hazards = [
    'BM-1'   => 8,
    'LT-1'   => 7,
    'LT-P1'  => 6,
    'LT-UNK' => 5,
    'BM-2'   => 4,
    'BM-3'   => 3,
    'BM-4'   => 2,
    'BM-U'   => 1,
    'NoGS'   => 0,
    'Not Screened' => 0,
];
 
$score = Score::all()->map(function($item) use ($hazards) {
    return $hazards[$item->field];
})->max();
```
The initial value could be this:

![image](https://user-images.githubusercontent.com/11309713/235352395-025aa892-c158-4278-9734-82e6068b89c3.png)


And your goal is to find the highest value from that array, from those values of the scores.

After map(), we return only the number from an array by score field.

![image](https://user-images.githubusercontent.com/11309713/235352406-38a23cfc-6e21-407a-9a92-359df1db9d25.png)


And the last max() only runs through all values and returns the maximum:
```
7
```
The GitHub repository with code can be found here. Inspiration source: Laracasts

## Example 3: pluck + flatten
Initial data comes from config which has a multi-dimentional array of settings. And our goal is to get the array of elements.

Code:
```
// config/setting_fields.php
return [
    'app' => [
        'title' => 'General',
        'desc'  => 'All the general settings for application.',
        'icon'  => 'fas fa-cube',
 
        'elements' => [
            [
                'type'  => 'text', // input fields type
                'data'  => 'string', // data type, string, int, boolean
                'name'  => 'app_name', // unique name for field
                'label' => 'App Name', // you know what label it is
                'rules' => 'required|min:2|max:50', // validation rule of laravel
                'class' => '', // any class for input
                'value' => 'Laravel Starter', // default value if you want
            ],
 
            // ...
        ],
    ],
    'email' => [
        'title' => 'Email',
        'desc'  => 'Email settings for app',
        'icon'  => 'fas fa-envelope',
 
        'elements' => [
            [
                'type'  => 'email', // input fields type
                'data'  => 'string', // data type, string, int, boolean
                'name'  => 'email', // unique name for field
                'label' => 'Email', // you know what label it is
                'rules' => 'required|email', // validation rule of laravel
                'class' => '', // any class for input
                'value' => 'info@example.com', // default value if you want
            ],
        ],
 
    ],
    'social' => [
        'title' => 'Social Profiles',
        'desc'  => 'Link of all the social profiles.',
        'icon'  => 'fas fa-users',
 
        'elements' => [
            [
                'type'  => 'text', // input fields type
                'data'  => 'string', // data type, string, int, boolean
                'name'  => 'facebook_url', // unique name for field
                'label' => 'Facebook Page URL', // you know what label it is
                'rules' => 'required|nullable|max:191', // validation rule of laravel
                'class' => '', // any class for input
                'value' => '#', // default value if you want
            ],
 
            // ...
        ],
 
        // ...
    ],
];
 
$elements = collect(config('setting_fields'))
    ->pluck('elements')
    ->flatten(1);
```
First, we make a collection for the array by doing collect(config('setting_fields')).

After the pluck() returns elements, those arrays form another collection element

![image](https://user-images.githubusercontent.com/11309713/235352440-a73e8dd4-6015-41f2-95de-01d48ec75183.png)


And then if we need to get rid of those indexes we can use flatten() and that will give us:

![image](https://user-images.githubusercontent.com/11309713/235352450-efce93bd-14df-469b-b5d0-6e5e96927ba1.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L294). Inspiration source: [nasirkhan/laravel-starter](https://github.com/nasirkhan/laravel-starter/blob/main/app/Models/Setting.php#L155).

## Example 4: filter + first & map + implode
Another example of two-method collection chains or in this case it's two chains by two methods. In the scenario, we have a few namespaces, where you can load Livewire components. And the goal is to have a class found in which namespace that is and then return the class name transformed into kebab case.

Code:
```
$class = 'App\\Base\\Http\\Livewire\\SomeClass';
$classNamespaces = [
    'App\\Base\\Http\\Livewire',
    'App\\Project\\Livewire'
];
 
$classNamespace = collect($classNamespaces)->filter(fn ($x) => strpos($class, $x) !== false)->first();
$namespace = collect(explode('.', str_replace(['/', '\\'], '.', $classNamespace)))
    ->map([Str::class, 'kebab'])
    ->implode('.');
```
First, we need to find which namespace corresponds to that class. We collect all namespaces and then filter by whether it contains that class or not. And then we get the first of the match.

Value after filter():

![image](https://user-images.githubusercontent.com/11309713/235352523-0bd04c06-1b70-4977-9f9a-721518bb521f.png)


Value after filter()->first():
```
"App\Base\Http\Livewire"
```
Then, we get that namespace, replace the slashes with the dot and explode that into an array and turn it into a new collection, and then map through that collection with a method. This is another way how you can use map(). Not only by providing a callback function, but providing a method from a class.

Value after filter()->first()->map():

![image](https://user-images.githubusercontent.com/11309713/235352553-267066fb-da07-4058-8bc4-c41a040980c9.png)


If any of those folder names are not corresponding to the kebab case they will be turned into a kebab case. In this case, it just goes into lowercase without any more transformations. And then we implode it back into one string:

Value after filter()->first()->map()->implode():
```
"app.base.http.livewire"
```
The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L253). Inspiration source: [iluminar/goodwork)](https://github.com/iluminar/goodwork/blob/master/app/Base/Utilities/ExtendedLivewireComponentsFinder.php).

## Example 5: push + map + implode
Now let's go one step higher and let's look at three methods of collection chains. A real-life example is the Twitter artisan giveaway command with the option to exclude some users.

Code:
```
$excluded = collect($this->option('exclude'))
    ->push('povilaskorop', '@dailylaravel')
    ->map(fn (string $name): string => str_replace('@', '', $name))
    ->implode(', ');
```
Call with parameters:
```
php artisan twitter:giveaway --exclude=someuser --exclude=@otheruser
```
This $this->option('exclude') is an array, and the initial value after making it into collection looks like this:

Initial value of collect($this->option('exclude')):

![image](https://user-images.githubusercontent.com/11309713/235352646-f8f54a62-9bc7-4bbd-b735-dfda115d845e.png)


When we do push() we add items to that collection.

Value after push():

![image](https://user-images.githubusercontent.com/11309713/235352664-f351f26a-80d0-4f94-bc49-eff9468a986c.png)


And then finally we do map() which is going through those items and replacing @ symbol with nothing.

Value after push()->map():

![image](https://user-images.githubusercontent.com/11309713/235352704-5f676920-7ccc-4df0-8c98-526e57ecd094.png)


And then we implode with a comma to provide the result in a visual format to be shown somewhere.

Value after push()->map()->implode():
```
"someuser, otheruser, povilaskorop, dailylaravel"
```
The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L38). Inspiration source: [Gummibeer/gummibeer.de](https://github.com/Gummibeer/gummibeer.de/blob/master/app/Console/Commands/TwitterGiveaway.php).

## Example 6: filter + map + implode
The next example is very similar but with a filter first and the example is different. So for example you have a User model with a lot of links to different social media profiles and you want to show those links as actual links separated by some parameter and also remove the empty ones.

Code:
```
$socialLinks = collect([
    'Twitter' => $user->link_twitter,
    'Facebook' => $user->link_facebook,
    'Instagram' => $user->link_instagram,
])
->filter()
->map(fn ($link, $network) => '<a href="' . $link . '">' . $network . '</a>')
->implode(' | ');
```
Initial value of collect():

![image](https://user-images.githubusercontent.com/11309713/235352761-e124bf1d-a747-44c3-9aad-5d71698ae7fb.png)


One value, in this case, Facebook is empty, that's why we need filter(). And filter() may have a parameter of callback function of what to filter by, or just filter() filters empty values.

Value after filter():

![image](https://user-images.githubusercontent.com/11309713/235352771-7af602b2-c66a-4953-9a23-9cdbffbbce07.png)


Then we map through those values and put them as links.

Value after filter()->map():

![image](https://user-images.githubusercontent.com/11309713/235352788-39a38531-eae2-4cde-a550-41394bdc5720.png)


And then we do a familiar implode with a vertical bar symbol as a separator and then we can put this string as a part of the blade file.

Value after filter()->map()->implode():
```
"<a href="https://twitter.com/povilaskorop">Twitter</a> | <a href="https://instagram.com/povilaskorop">Instagram</a>"
```
The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L45). Inspiration source: [spatie/freek.dev](https://github.com/spatie/freek.dev/blob/14a25e3b7f7c7d662e61746a9c24d28b9ac316b9/app/Models/Presenters/TalkPresenter.php).

## Example 7: reject + reject + each
The next example is repeating the same method twice. The task is you have a list of Models and you need to filter some objects out by some relationship condition, and then perform some job on each of them.

Code:
```
Repository::query()
    ->with('owner')
    ->get()
    ->reject(function (Repository $repository): bool {
        return $repository->owner instanceof User && $repository->owner->github_access_token === null;
    })
    ->reject(function (Repository $repository): bool {
        return $repository->owner instanceof Organization && $repository->owner->members()->whereIsRegistered()->doesntExist();
    })
    ->each(static function (Repository $repository): void {
        UpdateRepositoryDetails::dispatch($repository);
    });
```
Why use reject twice? Of course, we could use one reject() and then return condition and condition. But that would be more complicated to read. In this case, it's easier to read, reject the repository if the owner is User and User doesn't have a GitHub access token, or reject if the repository owner is Organization and members where is registered don't exist. It reads in plain English language.

Initial value of Repository::query()->with('owner')->get():

![image](https://user-images.githubusercontent.com/11309713/235352905-7eabe679-0111-46a1-8237-70e4b5d74334.png)


Value after first reject():

![image](https://user-images.githubusercontent.com/11309713/235352916-8b8851bb-f1e4-468a-ac5e-fb16f4319f36.png)


Value after second reject():

![image](https://user-images.githubusercontent.com/11309713/235352930-c01d5d4a-b930-481d-ba0a-b0f13836fc42.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L60). Inspiration source: [Astrotomic/opendor.me](https://github.com/Astrotomic/opendor.me/blob/dev/app/Console/Commands/GithubRepositoryDetails.php).

## Example 8: mapWithKeys + forget + filter
A few new methods in this example. The task here is that you have log files and then you need to identify older ones than X days.

Code:
```
$files = Storage::disk("logs")->allFiles();
$logFiles = collect($files)
    ->mapWithKeys(function ($file) {
        $matches = [];
        $isMatch = preg_match("/^laravel\-(.*)\.log$/i", $file, $matches);
 
        if (count($matches) > 1) {
            $date = $matches[1];
        }
 
        $key = $isMatch ? $date : "";
        return [$key => $file];
    })
    ->forget("")
    ->filter(function ($value, $key) use ($thresholdDate) {
        try {
            $date = Carbon::parse($key);
        } catch (\Exception $e) {
            return true;
        }
 
        return $date->isBefore($thresholdDate);
    });
```
We get all the log files and put them into the collection.

Initial value of collect($files):

![image](https://user-images.githubusercontent.com/11309713/235352979-a71f3d59-6ee5-46c8-a6ab-0ac73ea129fe.png)


Then we do a map but with keys. The goal is to return an array of [$date => $filename].

Value after mapWithKeys():

![image](https://user-images.githubusercontent.com/11309713/235352988-3612eae9-3f21-4b25-821e-b7e905c7c20a.png)


As you see there is one file without a date which we don't need to delete. For that, we use the forget() method.

Value after mapWithKeys()->forget():

![image](https://user-images.githubusercontent.com/11309713/235352996-4a54d9bf-a480-493c-9660-d6342f4165b4.png)


And finally, we do a filter of actually older files by key.

Value after mapWithKeys()->forget()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353012-1a64e7e0-e5c9-4055-b88a-de99d59942f3.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L147). Inspiration source: [opendialogai/opendialog](https://github.com/opendialogai/opendialog/blob/1.x/app/Console/Commands/ClearLogs.php).

## Example 9: map + filter + each
The next example is a typical kind of form for example or social network. You have a comment and some users mentioned the @username syntax. And from this string, you need to define those users and send a notification to them.

Code:
```
$comment = Comment::first();
collect($comment->mentionedUsers())
    ->map(function ($name) {
        return User::where('name', $name)->first();
    })
    ->filter()
    ->each(function ($user) use ($comment) {
        $user->notify(new YouWereMentionedNotification($comment));
    });
```
Initial value of $comment->description:
```
"I mention the @First user and the @Second user and @Third non-existing one."
```
First, we get mentioned users into a collection. The mentionedUsers() looks like this:
```
public function mentionedUsers()
{
    preg_match_all('/@([\w\-]+)/', $this->description, $matches);
    return $matches[1];
}
```
Initial value of $comment->mentionedUsers():

![image](https://user-images.githubusercontent.com/11309713/235353063-b8d38ab6-f7b0-40de-9803-85c85d950535.png)


Then we map through that collection and try to find a User with that name.

Value after map():

![image](https://user-images.githubusercontent.com/11309713/235353072-2614ff36-4c63-4b0b-870c-d123293a6238.png)


And then we filter null results.

Value after map()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353092-69574065-24fc-4ec8-969a-ffdffdef3c0d.png)


And then we can send to every User notification using the each() method.

The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L147). Inspiration source: [Bottelet/DaybydayCRM](https://github.com/Bottelet/DaybydayCRM/blob/d1965973f60933fd293aaaaa9f71e9d27edfd819/app/Listeners/NotiftyMentionedUsers.php).

## Example 10: map + filter + implode
We have a lot of categories and we need to get the slugs of those categories with active language, but the categories may have ancestors. It's a tree of categories.

Code:
```
$locale = 'en';
Category::all()
    ->map(function ($i) use ($locale) { return $i->getSlug($locale); })
    ->filter()
    ->implode('/');
```
Initial value of Category::all():

![image](https://user-images.githubusercontent.com/11309713/235353149-4b402d00-3dfb-4ada-a8ae-0f3807c5accb.png)


Then we map through categories to get slug. The getSlug() method looks like this:
```
public function getSlug($locale = null)
{
    if (($slug = $this->getActiveSlug($locale)) != null) {
        return $slug->slug;
    }
 
    if (config('translatable.use_property_fallback', false) && (($slug = $this->getFallbackActiveSlug()) != null)) {
        return $slug->slug;
    }
 
    return "";
}
 
public function getActiveSlug($locale = null)
{
    return $this->slugs->first(function ($slug) use ($locale) {
        return ($slug->locale === ($locale ?? app()->getLocale())) && $slug->active;
    }) ?? null;
}
 
public function getFallbackActiveSlug()
{
    return $this->slugs->first(function ($slug) {
        return $slug->locale === config('translatable.fallback_locale') && $slug->active;
    }) ?? null;
}
```
Value after map():

![image](https://user-images.githubusercontent.com/11309713/235353170-8f6c9b93-0d96-43f7-b531-b6bb5704c3e9.png)


Then we filter empty values.

Value after map()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353177-f797d3ca-2128-40ab-b63c-76522a2c5d01.png)


And then we implode with a slash to create a URL.

Value after map()->filter()->implode():
```
"first-category/second-category/third-category"
```
The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L239). Inspiration source: [area17/twill](https://github.com/area17/twill/blob/2.x/src/Models/Behaviors/HasNesting.php).

## Example 11: filter + filter + each
This example is pretty similar to earlier seen reject + reject. Here we have Post and we need to filter if Post is a Tweet. Then we need to filter that the external URL is empty. And then we fill external URL if we find twitter.com inside.

Code:
```
Post::all()
    ->filter->isTweet()
    ->filter(function (Post $post) {
        return empty($post->external_url);
    })
    ->each(function (Post $post) {
        preg_match('/(?=https:\/\/twitter.com\/).+?(?=")/', $post->text, $matches);
 
        if (count($matches) > 0) {
            $post->external_url = $matches[0];
            $post->save();
        }
    });
```
Here what is interesting is you can use filter like so filter->isTweet(). And that isTweet() method looks like this:
```
public function isTweet(): bool
{
    return $this->getType() === 'tweet';
}
 
public function getType(): string
{
    if ($this->hasTag('tweet')) {
        return 'tweet';
    }
 
    if ($this->original_content) {
        return 'original';
    }
 
    return 'link';
}
 
public function hasTag(string $tagName): bool
{
    return $this->tags->contains(fn (Tag $tag) => $tag->name === $tagName);
}
```
Initial value of Post::all():

![image](https://user-images.githubusercontent.com/11309713/235353242-4bc67cea-8170-4cdb-bd9f-4e815f8a1d17.png)


Now after the first filter isTweet():

![image](https://user-images.githubusercontent.com/11309713/235353259-09413e11-0de9-4391-af8d-f843a82e12d1.png)


One of them has an external URL null and another one has an external URL already filled. This is where another filter comes in to filter out Posts without external URLs.

![image](https://user-images.githubusercontent.com/11309713/235353277-89cc1113-6c66-41da-a13a-ef1c2b0acaa1.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L304). Inspiration source: [spatie/freek.dev](https://github.com/spatie/freek.dev/blob/main/app/Console/Commands/UpdateExternalUrlsWithTweetPermalinksCommand.php).

## Example 12: unique + filter + map + values
Here we have some events. Every event has a message, status, and subject. Then we need to get the unique messages, filter the ones that don't have a subject and extract the data into another structure

Code:
```
$events = Event::all();
$filteredEvents = $events
    ->unique(fn ($event) => $event->message)
    ->filter(fn ($event) => !is_null($event->subject))
    ->map(fn ($event) => $this->extractData($event))
    ->values();
```
Initial value of Event::all():

![image](https://user-images.githubusercontent.com/11309713/235353331-3c0bd12c-5889-4b62-8fd5-07d2a353904e.png)


Events with ID 1 and 2 are the same, so using unique() we remove duplicates from the collection.

Value after first unique():

![image](https://user-images.githubusercontent.com/11309713/235353340-eb6deac4-d7d5-400c-88a0-dbc269fd5346.png)


The next one is filter() which filters out the ones without the subject.

Value after ->unique()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353351-1556b3b9-be13-40e2-9fc1-3987fb1c412c.png)


The next one is map() which gives a different structure of data here.

Value after ->unique()->filter()->map():

![image](https://user-images.githubusercontent.com/11309713/235353358-84527642-7334-49c4-931e-0e5fca973294.png)


And the last one is to get the values and change the values of the array keys.

Value after ->unique()->filter()->map()->values():

![image](https://user-images.githubusercontent.com/11309713/235353367-a3760951-b556-44b8-8d1b-08b46212b6cb.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L82). Inspiration source: [opendialogai/opendialog](https://github.com/opendialogai/opendialog/blob/f7497ad94c21dfd78600e1a509f01067aaa0d323/app/Http/Responses/SelectionFrame.php).

## Example 13: map + mapToGroups + map + each
This example's goal is to log the last versions of Laravel and tell the users when was the last version when it is upgraded and stuff like that.

Code:
```
$versionsFromGithub = collect([
    ['name' => 'v9.19.0'],
    ['name' => 'v8.83.18'],
    ['name' => 'v9.18.0'],
    ['name' => 'v8.83.17'],
    ['name' => 'v9.17.0'],
    ['name' => 'v8.83.16'],
    // ...
]);
 
$versionsFromGithub
    // Map into arrays containing major, minor, and patch numbers
    ->map(function ($item) {
        $pieces = explode('.', ltrim($item['name'], 'v'));
 
        return [
            'major' => $pieces[0],
            'minor' => $pieces[1],
            'patch' => $pieces[2] ?? null,
        ];
    })
    // Map into groups by release; pre-6, major/minor pair; post-6, major
    ->mapToGroups(function ($item) {
        if ($item['major'] < 6) {
            return [$item['major'] . '.' . $item['minor'] => $item];
        }
 
        return [$item['major'] => $item];
    })
    // Take the highest patch or minor/patch number from each release
    ->map(function ($item) {
        if ($item->first()['major'] < 6) {
            // Take the highest patch
            return $item->sortByDesc('patch')->first();
        }
 
        // Take the highest minor, then its highest patch
        return $item->sortBy([['minor', 'desc'], ['patch', 'desc']])->first();
    })
    ->each(function ($item) {
        if ($item['major'] < 6) {
            $version = LaravelVersion::where([
                'major' => $item['major'],
                'minor' => $item['minor'],
            ])->first();
 
            if ($version->patch < $item['patch']) {
                $version->update(['patch' => $item['patch']]);
                info('Updated Laravel version ' . $version . ' to use latest patch.');
            }
        }
 
        $version = LaravelVersion::where([
            'major' => $item['major'],
        ])->first();
 
        if (! $version) {
            // Create it if it doesn't exist
            $created = LaravelVersion::create([
                'major' => $item['major'],
                'minor' => $item['minor'],
                'patch' => $item['patch'],
            ]);
 
            info('Created Laravel version ' . $created);
        }
        // Update the minor and patch if needed
        else if ($version->minor != $item['minor'] || $version->patch != $item['patch']) {
            $version->update(['minor' => $item['minor'], 'patch' => $item['patch']]);
            info('Updated Laravel version ' . $version . ' to use latest minor/patch.');
        }
    });
```
Initial values come from GitHub API and are put into the collection.

Initial value of $versionsFromGithub:

![image](https://user-images.githubusercontent.com/11309713/235353416-14342f93-e73d-421f-8b78-5a3e0f048c4d.png)


First, using map() we get major, minor, and patch versions, and remove the v from the start.

Value after map():

![image](https://user-images.githubusercontent.com/11309713/235353426-bc5ecd79-bf01-486d-910a-205d934dabb7.png)


Then we map into groups, where the group is the major version number.

Value after map()->mapToGroups():

![image](https://user-images.githubusercontent.com/11309713/235353446-c2108817-56a2-4db4-b68b-e18c63cb3c8a.png)


And then we use map() to return sorted results by minor and then by patch versions. And return only the first result which is the latest version.

Value after map()->mapToGroups()->map():

![image](https://user-images.githubusercontent.com/11309713/235353457-d0742d41-780c-454e-9e45-58ac81319484.png)


And then with each version, there's a huge operation that doesn't change the collection itself, but it checks if that version is in our database witch the major and minor versions. If it is, we update only the latest patch version. Otherwise, we create that Laravel version.

The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L162). Inspiration source: [tighten/laravelversions](https://github.com/tighten/laravelversions/blob/main/app/Console/Commands/FetchLatestReleaseNumbers.php).

## Example 14: map + flatten + map + filter
For example, we have two folders where the developer can put Livewire components. And we map through those folders to get the files. And the final result of that is the list of its items with a path name.

Code:
```
$folders = collect([
    "/Users/Povilas/Sites/project3/app/Base/Http/Livewire/",
    "/Users/Povilas/Sites/project3/app/Project/Livewire/"
]);
 
$classNames = $folders
    ->map(function ($item) {
        return (new Filesystem())->allFiles($item);
    })
    ->flatten()
    ->map(function (\SplFileInfo $file) {
        return app()->getNamespace().str_replace(
                ['/', '.php'],
                ['\\', ''],
                Str::after($file->getPathname(), app_path().'/')
            );
    })
    ->filter(function (string $class) {
        return is_subclass_of($class, Component::class) &&
            ! (new \ReflectionClass($class))->isAbstract();
    });
```
Value after map():

![image](https://user-images.githubusercontent.com/11309713/235353521-7e0a4198-4ea4-4867-8eaa-55b9ac1d4c76.png)


Then we do flatten() without any parameters. We flatten all the files into one level.

Value after map()->flatten():

![image](https://user-images.githubusercontent.com/11309713/235353532-65b834a6-5d80-46a5-b0a5-922dc15aee0e.png)


And then we map to get the data we want.

Value after map()->flatten()->map():

![image](https://user-images.githubusercontent.com/11309713/235353543-629591fd-2a45-4e1e-b3ab-31e99cb2fd8a.png)


And the last filter if the class is a Livewire Component and not an abstract component.

Value after map()->flatten()->map()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353552-64676c04-5b24-4665-b9de-ccb5bb96c15e.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/cf8469ff2febe296efa399ae85659a6d6dc4e1bf/app/Http/Controllers/ExampleController.php#L326). Inspiration source: [iluminar/goodwork](https://github.com/iluminar/goodwork/blob/master/app/Base/Utilities/ExtendedLivewireComponentsFinder.php).

## Example 15: mapWithKeys + each + reject + filter + all
This example gets all packages from the composer and filters them out in various ways. Here the goal is only to have Laravel packages. It is saved in $package['extra']['laravel'].

Code:
```
if ($filesystem->exists($path = base_path() . '/vendor/composer/installed.json')) {
    $plugins = json_decode($filesystem->get($path), true);
}
 
$packages = collect($plugins['packages'])
    ->mapWithKeys(function ($package) {
        return [$this->format($package['name']) => $package['extra']['laravel'] ?? []];
    })
    ->each(function ($configuration) use (&$ignore) {
        $ignore = array_merge($ignore, $configuration['dont-discover'] ?? []);
    })
    ->reject(function ($configuration, $package) use ($ignore) {
        return in_array($package, $ignore);
    })
    ->filter()
    ->all();
```
Initial value of $plugins['packages']:

![image](https://user-images.githubusercontent.com/11309713/235353576-5c250279-c34c-41fa-b057-3aa89cab7667.png)


First, we simplify the array with mapWithKeys(). We only need the package name and ['extra']['laravel'] values.

Value after mapWithKeys():

![image](https://user-images.githubusercontent.com/11309713/235353587-3db28801-0991-44cc-a81a-cfb6e61ddde8.png)


The next move is to remove packages with dont-discover, and then reject() all those packages from the list. Now we have fewer packages.

Value after mapWithKeys()->each()->reject():

![image](https://user-images.githubusercontent.com/11309713/235353600-f1cd788c-0cde-447f-abee-5e07960e0a30.png)


But we have a lot of packages with empty parameters and the filter() method without any parameters removes empty ones from the collection.

Value after mapWithKeys()->each()->reject()->filter():

![image](https://user-images.githubusercontent.com/11309713/235353612-ad63ced8-cd6f-46ec-b0fd-20072757726e.png)


And finally, we are doing all() which transforms the collection into an array.

Value after mapWithKeys()->each()->reject()->filter()->all():

![image](https://user-images.githubusercontent.com/11309713/235353625-1c9f7366-ad50-4e9b-b6b7-0ed3fc98d044.png)


The GitHub repository with code can be [found here](https://github.com/LaravelDaily/Laravel-Collections-Chains-Demos/blob/main/app/Http/Controllers/ExampleController.php#L358). Inspiration source: [iluminar/goodwork](https://github.com/iluminar/goodwork/blob/92e685c8bc0ce954ff4ff4d72a7c90a2b38d0415/app/Base/Utilities/PluginManifest.php#L43).

That's it!
I hope these examples will give you the ideas how you can use Collections in your applications, especially in the chained way.
