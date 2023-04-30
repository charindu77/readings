# Livewire Like/Dislike Component for Social Networks: Step-by-Step
In this tutorial, we will use Livewire to create a component for Like/Dislike, similar to YouTube or any social network. We will show the count of likes and dislikes, also minimizing the number of queries to the DB.

![image](https://user-images.githubusercontent.com/11309713/235369354-785aaec2-a286-4da8-98e6-96e5f52adb1f.png)

## Laravel Project Preparation
We'll show the list of posts and the number of votes for every post.

For this demo, we'll use our own Laravel Breeze Pages Skeleton which will give use Breeze-like layout but with a public page for posts list.

First, we need a Post Model, Controller, and View. For now, without any Livewire.
```
php artisan make:model Post -mc
```
*database/migrations/xxxx_create_posts_table.php:*
```
return new class extends Migration {
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('text');
            $table->timestamps();
        });
    }
};
```
app/Models/Post.php:
```
class Post extends Model
{
    protected $fillable = [
        'title',
        'text',
    ];
}
```
app/Http/Controllers/PostController.php:
```
class PostController extends Controller
{
    public function __invoke(): View
    {
        $posts = Post::latest()->paginate();
 
        return view('posts', compact('posts'));
    }
}
```
And a simple Blade file to show the list of the posts.

resources/views/posts.blade.php:
```
<x-app-layout>
    <x-slot name="header">
        <h2 class="text-xl font-semibold leading-tight text-gray-800 dark:text-gray-200">
            {{ __('Posts') }}
        </h2>
    </x-slot>
 
    <div class="py-12">
        <div class="mx-auto max-w-7xl sm:px-6 lg:px-8">
            <div class="overflow-hidden bg-white shadow-sm dark:bg-gray-800 sm:rounded-lg">
                <div class="p-6 text-gray-900 dark:text-gray-100">
                    @foreach($posts as $post)
                        <h3 class="text-xl font-medium">{{ $post->title }}</h3>
                        <p>{{ $post->text }}</p>
                        <hr class="my-4">
                    @endforeach
 
                    {{ $posts->links() }}
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```
![image](https://user-images.githubusercontent.com/11309713/235369425-a5a8dc12-c582-4549-a657-ef866f352855.png)

Next, we need to save votes. For this, we will create a Vote Model.
```
php artisan make:model Vote -m
```
database/migrations/xxxx_create_votes_table.php:
```
return new class extends Migration {
    public function up(): void
    {
        Schema::create('votes', function (Blueprint $table) {
            $table->id();
            $table->foreignId('post_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->smallInteger('vote');
            $table->timestamps();
        });
    }
};
```
app/Models/Vote.php:
```
class Vote extends Model
{
    protected $fillable = [
        'post_id',
        'user_id',
        'vote',
    ];
}
```
Now let's add Vote relations to the Post Model.

app/Models/Post.php:
```
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Database\Eloquent\Relations\HasMany;
 
class Post extends Model
{
    protected $fillable = [
        'title',
        'text',
    ];
 
    public function votes(): HasMany 
    {
        return $this->hasMany(Vote::class);
    } 
 
    public function userVotes(): HasOne 
    {
        return $this->votes()->one()->where('user_id', auth()->id());
    } 
}
```
We'll need two relations. The first one is just a regular One To Many relation which we will use to create and update the vote for the post.

The second one is more interesting: the userVotes relation will return null if a user hasn't voted yet. Otherwise, it will return the Vote model from which we will be able to tell if a user liked or disliked a post.

## Livewire Component
Now let's create a Livewire component.
```
php artisan make:livewire LikeDislike
```
First, let's add the Livewire component after the post text and show like and dislike buttons. For buttons, I will use heroicons SVG.

resources/views/posts.blade.php:
```
// ...
@foreach($posts as $post)
    <h3 class="text-xl font-medium">{{ $post->title }}</h3>
    <p>{{ $post->text }}</p>
    @livewire('like-dislike', [$post]) 
    <hr class="my-4">
@endforeach
// ...
</x-app-layout>
```
resources/views/livewire/like-dislike.blade.php:
```
<div class="mt-1">
    <div class="inline-flex rounded-md bg-gray-100 px-2 py-1 space-x-2">
        <div class="flex">
            <a wire:click="like()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 mr-1 rounded hover:bg-gray-300">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6.633 10.5c.806 0 1.533-.446 2.031-1.08a9.041 9.041 0 012.861-2.4c.723-.384 1.35-.956 1.653-1.715a4.498 4.498 0 00.322-1.672V3a.75.75 0 01.75-.75A2.25 2.25 0 0116.5 4.5c0 1.152-.26 2.243-.723 3.218-.266.558.107 1.282.725 1.282h3.126c1.026 0 1.945.694 2.054 1.715.045.422.068.85.068 1.285a11.95 11.95 0 01-2.649 7.521c-.388.482-.987.729-1.605.729H13.48c-.483 0-.964-.078-1.423-.23l-3.114-1.04a4.501 4.501 0 00-1.423-.23H5.904M14.25 9h2.25M5.904 18.75c.083.205.173.405.27.602.197.4-.078.898-.523.898h-.908c-.889 0-1.713-.518-1.972-1.368a12 12 0 01-.521-3.507c0-1.553.295-3.036.831-4.398C3.387 10.203 4.167 9.75 5 9.75h1.053c.472 0 .745.556.5.96a8.958 8.958 0 00-1.302 4.665c0 1.194.232 2.333.654 3.375z" />
                </svg>
            </a>
            0
        </div>
        <div class="flex">
            <a wire:click="dislike()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 mr-1 rounded hover:bg-gray-300">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M7.5 15h2.25m8.024-9.75c.011.05.028.1.052.148.591 1.2.924 2.55.924 3.977a8.96 8.96 0 01-.999 4.125m.023-8.25c-.076-.365.183-.75.575-.75h.908c.889 0 1.713.518 1.972 1.368.339 1.11.521 2.287.521 3.507 0 1.553-.295 3.036-.831 4.398C20.613 14.547 19.833 15 19 15h-1.053c-.472 0-.745-.556-.5-.96a8.95 8.95 0 00.303-.54m.023-8.25H16.48a4.5 4.5 0 01-1.423-.23l-3.114-1.04a4.5 4.5 0 00-1.423-.23H6.504c-.618 0-1.217.247-1.605.729A11.95 11.95 0 002.25 12c0 .434.023.863.068 1.285C2.427 14.306 3.346 15 4.372 15h3.126c.618 0 .991.724.725 1.282A7.471 7.471 0 007.5 19.5a2.25 2.25 0 002.25 2.25.75.75 0 00.75-.75v-.633c0-.573.11-1.14.322-1.672.304-.76.93-1.33 1.653-1.715a9.04 9.04 0 002.86-2.4c.498-.634 1.226-1.08 2.032-1.08h.384"/>
                </svg>
            </a>
            0
        </div>
    </div>
</div>
```
Now after every post, we should see two buttons with hard-coded 0 count values.

Also, both buttons now have actions for wire:click="like()" and wire:click="dislike()" respectfully. We will implement them in a minute.

![image](https://user-images.githubusercontent.com/11309713/235369470-93c8cd2f-e123-4a6f-84fd-6dce7bf55ea8.png)

For the component, we passed the Post model.

app/Http/Livewire/LikeDislike.php:
```
use App\Models\Post;
 
class LikeDislike extends Component
{
    public Post $post; 
 
    public function mount(Post $post): void 
    {
        $this->post = $post;
    } 
 
    public function render(): View
    {
        return view('livewire.like-dislike');
    }
}
```
Next, validation for non-logged-in users. We allow only registered users to hit like/dislike, right?

When a guest tries to click any of the buttons, we need to show an error.

For this, we'll make a private validateAccess() method in the Livewire component and will call it on both like() and dislike() methods.

app/Http/Livewire/LikeDislike.php:
```
use Illuminate\Validation\ValidationException;
 
class LikeDislike extends Component
{
    // ...
    public function like(): void
    {
        $this->validateAccess(); 
    }
 
    public function dislike(): void
    {
        $this->validateAccess(); 
    }
 
    public function render(): View
    {
        return view('livewire.like-dislike');
    }
 
    private function validateAccess(): bool 
    {
        throw_if(
            auth()->guest(),
            ValidationException::withMessages(['unauthenticated' => 'You need to <a href="' . route('login') . '" class="underline">login</a> to click like/dislike'])
        );
    } 
}
```
And we need to show the error message in the blade.
```
resources/views/livewire/like-dislike.blade.php:

<div class="mt-1">
    // ...
    @error('unauthenticated') 
        <div class="text-red-500">{!! $message !!}</div>
    @enderror 
</div>
```
After clicking any button now guest will see an error message like below:

![image](https://user-images.githubusercontent.com/11309713/235369490-1283e9dc-1c3c-4827-9178-5f83faf994d6.png)

## Saving Like or Dislike
In the Vote table we will save the vote value in the vote field, where 1 will mean liked and -1 disliked.

We will also set the value to 0 if a user removes any of the options.

First, we need two new properties:

- $userVote: we will store the user vote from the userVotes relation
- $lastUserVote: the last user vote value taken from the same relation
app/Http/Livewire/LikeDislike.php:
```
use App\Models\Post;
 
class LikeDislike extends Component
{
    public Post $post;
    public ?Vote $userVote = null; 
    public int $lastUserVote = 0; 
 
    public function mount(Post $post): void
    {
        $this->post = $post;
        $this->userVote = $post->userVotes; 
        $this->lastUserVote = $this->userVote->vote ?? 0; 
    }
    // ...
}
```
But this way we will get an N+1 Query problem and performance issues.

To avoid it, we need to eager load the userVotes relation in the Controller.

app/Http/Controllers/PostController.php:
```
class PostController extends Controller
{
    public function __invoke(): View
    {
        $posts = Post::with('userVotes') 
            ->latest()
            ->paginate();
 
        return view('posts', compact('posts'));
    }
}
```
Now, for the convenience of checking if a user has liked/disliked, we will create a method hasVoted to reuse it.

app/Http/Livewire/LikeDislike.php:
```
class LikeDislike extends Component
{
    // ...
 
    public function like(): void
    {
        $this->validateAccess();
 
        if ($this->hasVoted(1)) {
            // TODO: update vote
        }
 
        // TODO: update vote
    }
 
    public function dislike(): void
    {
        $this->validateAccess();
 
        if ($this->hasVoted(-1)) {
            // TODO: update vote
        }
 
        // TODO: update vote
    }
 
    public function render(): View
    {
        return view('livewire.like-dislike');
    }
 
    private function hasVoted(int $val): bool 
    {
        return $this->userVote && $this->userVote->vote === $val;
    } 
}
```
And now, because we have the last user-voted value we can add a background to the SVG to show the user what he has chosen.

resources/views/livewire/like-dislike.blade.php:
```
<div class="mt-1">
    <div class="inline-flex rounded-md bg-gray-100 px-2 py-1 space-x-2">
        <div class="flex">
            <a wire:click="like()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 mr-1 rounded hover:bg-gray-300"> 
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" @class(['w-6 h-6 mr-1 rounded hover:bg-gray-300', 'bg-gray-300' => $lastUserVote === 1])> {{--  ++}}
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6.633 10.5c.806 0 1.533-.446 2.031-1.08a9.041 9.041 0 012.861-2.4c.723-.384 1.35-.956 1.653-1.715a4.498 4.498 0 00.322-1.672V3a.75.75 0 01.75-.75A2.25 2.25 0 0116.5 4.5c0 1.152-.26 2.243-.723 3.218-.266.558.107 1.282.725 1.282h3.126c1.026 0 1.945.694 2.054 1.715.045.422.068.85.068 1.285a11.95 11.95 0 01-2.649 7.521c-.388.482-.987.729-1.605.729H13.48c-.483 0-.964-.078-1.423-.23l-3.114-1.04a4.501 4.501 0 00-1.423-.23H5.904M14.25 9h2.25M5.904 18.75c.083.205.173.405.27.602.197.4-.078.898-.523.898h-.908c-.889 0-1.713-.518-1.972-1.368a12 12 0 01-.521-3.507c0-1.553.295-3.036.831-4.398C3.387 10.203 4.167 9.75 5 9.75h1.053c.472 0 .745.556.5.96a8.958 8.958 0 00-1.302 4.665c0 1.194.232 2.333.654 3.375z" />
                </svg>
            </a>
            0
        </div>
        <div class="flex">
            <a wire:click="dislike()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 mr-1 rounded hover:bg-gray-300"> {{--  --}}
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" @class(['w-6 h-6 mr-1 rounded hover:bg-gray-300', 'bg-gray-300' => $lastUserVote === -1])> 
                    <path stroke-linecap="round" stroke-linejoin="round" d="M7.5 15h2.25m8.024-9.75c.011.05.028.1.052.148.591 1.2.924 2.55.924 3.977a8.96 8.96 0 01-.999 4.125m.023-8.25c-.076-.365.183-.75.575-.75h.908c.889 0 1.713.518 1.972 1.368.339 1.11.521 2.287.521 3.507 0 1.553-.295 3.036-.831 4.398C20.613 14.547 19.833 15 19 15h-1.053c-.472 0-.745-.556-.5-.96a8.95 8.95 0 00.303-.54m.023-8.25H16.48a4.5 4.5 0 01-1.423-.23l-3.114-1.04a4.5 4.5 0 00-1.423-.23H6.504c-.618 0-1.217.247-1.605.729A11.95 11.95 0 002.25 12c0 .434.023.863.068 1.285C2.427 14.306 3.346 15 4.372 15h3.126c.618 0 .991.724.725 1.282A7.471 7.471 0 007.5 19.5a2.25 2.25 0 002.25 2.25.75.75 0 00.75-.75v-.633c0-.573.11-1.14.322-1.672.304-.76.93-1.33 1.653-1.715a9.04 9.04 0 002.86-2.4c.498-.634 1.226-1.08 2.032-1.08h.384"/>
                </svg>
            </a>
            0
        </div>
    </div>
    @error('unauthenticated')
        <div class="text-red-500">{!! $message !!}</div>
    @enderror
</div>
```
If the user liked or disliked a post, they will see a different background.

![image](https://user-images.githubusercontent.com/11309713/235369547-48e8729c-6821-492e-8700-69ff4379a18d.png)

Now, when setting the vote value, we need to check if the user has already voted.

We have set this in the $userVote property. If the user has voted we need to update the vote value. Otherwise, we need to create a new record.

We'll do this in a new method updateVote and after the DB operations, we'll set the $lastUserVote property to the new value.

app/Http/Livewire/LikeDislike.php:
```
class LikeDislike extends Component
{
    // ...
    public function like()
    {
        $this->validateAccess();
 
        if ($this->hasVoted(1)) {
            $this->updateVote(0); 
            return; 
        }
 
        $this->updateVote(1); 
    }
 
    public function dislike(): void
    {
        $this->validateAccess();
 
        if ($this->hasVoted(-1)) {
            $this->updateVote(0); 
            return; 
        }
 
        $this->updateVote(-1); 
    }
    // ...
    private function updateVote(int $val): void 
    {
        if ($this->userVote) {
            $this->post->votes()->update(['user_id' => auth()->id(), 'vote' => $val]);
        } else {
            $this->userVote = $this->post->votes()->create(['user_id' => auth()->id(), 'vote' => $val]);
        }
 
        $this->lastUserVote = $val;
    } 
    // ...
}
```
## Show Count of Likes and Dislikes
In this part, we will show how many likes and dislikes the post has. Also, after the user votes, we'll update that count.

First, in the Controller, we need to get the count and set it to public property in the Livewire component.

app/Http/Controllers/PostController.php:
```
use Illuminate\Database\Eloquent\Builder;
 
class PostController extends Controller
{
    public function __invoke(): View
    {
        $posts = Post::with('userVotes')
            ->withCount(['votes as likesCount' => fn (Builder $query) => $query->where('vote', '>', 0)], 'vote') 
            ->withCount(['votes as dislikesCount' => fn (Builder $query) => $query->where('vote', '<', 0)], 'vote') 
            ->latest()
            ->paginate();
 
        return view('posts', compact('posts'));
    }
}
```
app/Http/Livewire/LikeDislike.php:
```
class LikeDislike extends Component
{
    public Post $post;
    public ?Vote $userVote = null;
    public int $likes = 0; 
    public int $dislikes = 0; 
    public int $lastUserVote = 0;
 
    public function mount(Post $post): void
    {
        $this->post = $post;
        $this->userVote = $post->userVotes;
        $this->likes = $post->likesCount; 
        $this->dislikes = $post->dislikesCount; 
        $this->lastUserVote = $this->userVote->vote ?? 0;
    }
    // ...
}
```
Now we can change the hard-coded "0" values to the real ones, in the Blade file.

resources/views/livewire/like-dislike.blade.php:
```
<div class="mt-1">
    <div class="inline-flex rounded-md bg-gray-100 px-2 py-1 space-x-2">
        <div class="flex">
            <a wire:click="like()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" @class(['w-6 h-6 mr-1 rounded hover:bg-gray-300', 'bg-gray-300' => $lastUserVote === 1])>
                    <path stroke-linecap="round" stroke-linejoin="round" d="M6.633 10.5c.806 0 1.533-.446 2.031-1.08a9.041 9.041 0 012.861-2.4c.723-.384 1.35-.956 1.653-1.715a4.498 4.498 0 00.322-1.672V3a.75.75 0 01.75-.75A2.25 2.25 0 0116.5 4.5c0 1.152-.26 2.243-.723 3.218-.266.558.107 1.282.725 1.282h3.126c1.026 0 1.945.694 2.054 1.715.045.422.068.85.068 1.285a11.95 11.95 0 01-2.649 7.521c-.388.482-.987.729-1.605.729H13.48c-.483 0-.964-.078-1.423-.23l-3.114-1.04a4.501 4.501 0 00-1.423-.23H5.904M14.25 9h2.25M5.904 18.75c.083.205.173.405.27.602.197.4-.078.898-.523.898h-.908c-.889 0-1.713-.518-1.972-1.368a12 12 0 01-.521-3.507c0-1.553.295-3.036.831-4.398C3.387 10.203 4.167 9.75 5 9.75h1.053c.472 0 .745.556.5.96a8.958 8.958 0 00-1.302 4.665c0 1.194.232 2.333.654 3.375z" />
                </svg>
            </a>
            0 
            {{ $likes }} 
        </div>
        <div class="flex">
            <a wire:click="dislike()" class="cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" @class(['w-6 h-6 mr-1 rounded hover:bg-gray-300', 'bg-gray-300' => $lastUserVote === -1])>
                    <path stroke-linecap="round" stroke-linejoin="round" d="M7.5 15h2.25m8.024-9.75c.011.05.028.1.052.148.591 1.2.924 2.55.924 3.977a8.96 8.96 0 01-.999 4.125m.023-8.25c-.076-.365.183-.75.575-.75h.908c.889 0 1.713.518 1.972 1.368.339 1.11.521 2.287.521 3.507 0 1.553-.295 3.036-.831 4.398C20.613 14.547 19.833 15 19 15h-1.053c-.472 0-.745-.556-.5-.96a8.95 8.95 0 00.303-.54m.023-8.25H16.48a4.5 4.5 0 01-1.423-.23l-3.114-1.04a4.5 4.5 0 00-1.423-.23H6.504c-.618 0-1.217.247-1.605.729A11.95 11.95 0 002.25 12c0 .434.023.863.068 1.285C2.427 14.306 3.346 15 4.372 15h3.126c.618 0 .991.724.725 1.282A7.471 7.471 0 007.5 19.5a2.25 2.25 0 002.25 2.25.75.75 0 00.75-.75v-.633c0-.573.11-1.14.322-1.672.304-.76.93-1.33 1.653-1.715a9.04 9.04 0 002.86-2.4c.498-.634 1.226-1.08 2.032-1.08h.384"/>
                </svg>
            </a>
            0 
            {{ $dislikes }} 
        </div>
    </div>
    @error('unauthenticated')
        <div class="text-red-500">{!! $message !!}</div>
    @enderror
</div>
```
All that is left, is to update the count numbers, in the updateVote method.

To make that method less bloated, we will create a new method just for updating the count and will call it setLikesAndDislikesCount.

In this method we will use the match expression to check the value of lastUserVote and the new value and increase/decrease the count of likes and dislikes.

app/Http/Livewire/LikeDislike.php:
```
class LikeDislike extends Component
{
    // ...
    private function updateVote(int $val): void
    {
        if ($this->userVote) {
            $this->post->votes()->update(['user_id' => auth()->id(), 'vote' => $val]);
        } else {
            $this->userVote = $this->post->votes()->create(['user_id' => auth()->id(), 'vote' => $val]);
        }
        $this->setLikesAndDislikesCount($val); 
 
        $this->lastUserVote = $val;
    }
 
    // ...
 
    private function setLikesAndDislikesCount(int $val): void 
    {
        match (true) {
            $this->lastUserVote === 0 && $val === 1 => $this->likes++,
            $this->lastUserVote === 0 && $val === -1 => $this->dislikes++,
            $this->lastUserVote === 1 && $val === 0 => $this->likes--,
            $this->lastUserVote === -1 && $val === 0 => $this->dislikes--,
            $this->lastUserVote === -1 && $val === 1 => call_user_func(function () {
                $this->dislikes--;
                $this->likes++;
            }),
            $this->lastUserVote === 1 && $val === -1 => call_user_func(function () {
                $this->dislikes++;
                $this->likes--;
            }),
        };
    } 
}
```
It may look overcomplicated, but this way we avoid querying the DB so end up with better performance.

All the code used in this tutorial you can find in the [GitHub repository here.](https://github.com/LaravelDaily/Livewire-Like-Dislike-YouTube)
