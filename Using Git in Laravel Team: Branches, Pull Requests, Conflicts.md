# Using Git in Laravel Team: Branches, Pull Requests, Conflicts
Git is an essential tool for every developer. In this tutorial, I will explain everything you need to know about branches and conflicts while working in a team, with Laravel examples.

In fact, this article is not about Laravel, it's just that example code will be with Laravel framework, but you can apply Git knowledge from here to other coding languages/frameworks, too.

Notice: in this tutorial, we will use Terminal and not use visual tools like Sourcetree/GitHub Desktop or PhpStorm/VSCode editors. Those tools can help but I want you to learn principles and the syntax if you do need to work with the Terminal.

Start with Branches: Main/Master and Develop
Branches are probably the foundation when working with the team. But even when working solo, you may use branches to work on separate features simultaneously, so you may have branches called "feature-payment", "feature-user-update", etc.

Also, you may use them for testing features from branches without deploying them to live. So, your testing server would get the code from "develop" branch, for example, and you can play around there, until you're sure it works, and then merge the code into the "main" branch for deployment.

Every team choose their own branch names and branch logic, but there's a most typical behavior, here's how it goes.

When the repository is created, it is created with the branch called main.

Notice: GitHub changed its default from master to main for new repositories in 2020.

git main branch

Next, let's create a new fresh Laravel project and push its code to GitHub.

laravel new laravel
cd laravel

Now we can push to that main branch:

git init
git add .
git commit -m "Fresh Laravel"
git branch -M main
git remote add origin git@github.com:krekas/Git-Example.git
git push -u origin main

What every command here does?

Creates an empty Git repository.
Adds all files to the repository.
Makes a commit with the message "Fresh Laravel".
Sets branch to main.
Adds an origin where to push.
Pushes code to the main branch.
If you refresh the GitHub repository page, you will see the code was pushed to the main branch.

fresh code in the main branch

Next step: usually the main branch is only for the "finalized" features to deploy to live. And the work is being done in other branches. So, from that main branch, someone creates a develop branch where the work is being done. Here's how you can do it on GitHub directly in the browser.

create develop branch

At this point, the develop code is identical to the main branch.

Then, whenever a developer starts working with the project, they clone the repository, checkout the develop branch and starts the work.

The main branch is only for deployment to the live server which happens from time to time, but more rarely than everyday work.

You can even protect the branches on GitHub, to restrict pushing to the main branch, to avoid accidentally pushing wrong things to live server.

Routine Work: Checkout Develop and Feature Branches
At any point, you are working with one specific branch. To check what branch are you on, you do git status.

On branch main
Your branch is up to date with 'origin/main'.
 
nothing to commit, working tree clean

And you see On branch main which means that you are on the main branch. Next, you need to do git fetch: it downloads information about newer changes (but not the code itself, yet). And we see that the new change is about a new branch.

From github.com:krekas/Git-Example
 * [new branch]      develop    -> origin/develop

Now you can switch into the develop branch by doing git checkout develop.

Branch 'develop' set up to track remote branch 'develop' from 'origin'.
Switched to a new branch 'develop'

The develop branch code will be downloaded at this point.

So this is how you need to start with every new feature: checking out the latest version of develop and start coding.

From here, there are two ways how teams agree to work:

Work directly on the develop branch: generally, it happens when there's only one developer and no team
Or create a thing called feature branches. So whenever someone starts a new feature, they need to create their branch, branching from develop, naming the branch accordingly to the feature (like "feature-payments"), and then whenever they are ready they need to merge into the develop branch, not into main.
Personally, I recommend using feature branches even if you're solo, because it allows you to work on multiple features at the same time, choosing which one(s) to merge/push into the next live deployment.

The main branch still remains kinda like a sacred thing where only the repository owner or person who will deploy the code is responsible for this branch. Everyone else works on develop or feature branches.

Finished the Task? Pull Request from Feature Branch
Let's look at a real scenario how work would happen.

Imagine we have a task to install Laravel Breeze. First, we need to create (checkout) the new feature branch.

git checkout -b breeze-install                                                                                                                                               -╯
Switched to a new branch 'breeze-install'

This command does two things:

creates the new branch
switches to it
So our code changes will be saved on the breeze-install branch and not develop, until we decide to push and merge into develop.

Now, let's install the package.

composer require laravel/breeze --dev
php artisan breeze:install blade

Ok, so our code task is done. Now let's push new code to the feature branch.

git add .
git commit -m "Laravel Breeze"
git push --set-upstream origin breeze-install

And now in the GitHub repository, we have three branches.

three branches in the repository

The next step is to create a thing called Pull Request. This is a request for other team members (usually your senior developer) to check that code and approve that it should go into the develop branch.

GitHub web version can help you with suggestions for the pull request. So you can click Compare & pull request.

github offers pr

Name the pull request with the feature name. And the most important part is to change where to merge to the develop branch.

pr to develop branch

It will check if there are any code conflicts, and if you see Able to merge you can just Create pull request.

And that's it for regular developer workflow: the task is over for now until it gets checked. Next, someone who is responsible for the code review will check it, and if everything is correct, it will be merged into the develop, and later deployed to server(s).

Or, if the teammates ask you to make changes in your code, you do that in the same feature branch, just do git push and your changes will automatically applied to the same Pull Request.

Avoid Conflicts with Feature Branches
Often you have a situation: you work on one feature, then you make a Pull Request, and then until someone approves that, you need to start a new feature. So how to avoid conflicts?

Let's demonstrate it be working a new feature: for example, add a surname to the user table.

First, a new branch should be made from the develop branch.

git checkout develop
git checkout -b feature/user-surname

Next, quickly we add the surname.

php artisan make:migration "add surname to users table"

database/migrations/xxxx_add_surname_to_users_table.php

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('surname');
        });
    }
};

app/Models/User.php:

class User extends Authenticatable
{
    // ...
    protected $fillable = [
        'name',
        'email',
        'password',
        'surname', 
    ];
    // ...
}

And now we are ready to commit the code.

With git status we can check what files were changed, and two files are changed:

On branch feature/user-surname
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
    modified:   app/Models/User.php
 
Untracked files:
  (use "git add <file>..." to include in what will be committed)
    database/migrations/2023_03_26_140758_add_surname_to_users_table.php
 
no changes added to commit (use "git add" and/or "git commit -a")

Now let's add files and commit:

git add .
git commit -m "Surname for user"
git push --set-upstream origin feature/user-surname

Now we are ready to create a Pull Request.

Go to GitHub, create a Pull Request, and don't forget to change the base branch to the develop branch.

create pr

Now, what happens until someone approves this feature? We receive a new task also related to that new user, what do we do?

A typical mistake that I saw from junior developers or from those who are new to git, is to continue working on the same feature branch.

That's why those branches are called feature branches, because every branch is related to only one feature. So if you have a new feature, for example adding another field into the user's table, then you need to check out the develop branch again, then create another feature branch from develop and continue on that separate branch.

While approving the first task, the person who is approving that will have a much better experient if you stick to only one feature and the code in the pull request will be related only to that feature. It will be easier to read, understand, test and comment/approve.

If you add unrelated code to the same branch, then it will be automatically added to that pull request and will confuse the person who is approving. So one feature branch, only code for that branch, and you continue.

Next, let's create another pull request. First, we need to switch to the develop branch, and only then create a new feature branch.

git checkout develop
git checkout -b feature/verify-email

And quickly let's add the MustVerifyEmail interface to the User Model.

app/Models/User.php:

// use Illuminate\Contracts\Auth\MustVerifyEmail; // 
use Illuminate\Contracts\Auth\MustVerifyEmail; 
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
 
class User extends Authenticatable 
class User extends Authenticatable implements MustVerifyEmail 
{
    // ...
}

Now, we need to commit and make a pull request, don't forget to set the base branch to develop:

git add .
git commit -m "Verify user email"
git push --set-upstream origin feature/verify-email

And now we have two opened Pull Requests.

two opened prs

If we merge the first Pull Request, with the feature of adding a surname.

closed first pr

And in the second feature pull request, we can see changes that are made only for this feature. The person who will review it will be able to quickly merge.

feature pr

Pull Down Develop Before Starting New Feature
Another tip to avoid conflicts.

Imagine a scenario you have to add a new field to the User Model. You add it, then make a Pull Request and call it a day.

github pr

Later reviewer merges this Pull Request. The next day you continue on this project with a new feature. Let's say you have to add the country field to the same User model.

You would checkout into the develop branch and create a new branch for a new feature like git checkout -b feature/user-country.

But STOP. In this case, a golden rule is before starting a new feature you have to pull down the changes from develop. Especially when working in a team there's a big chance that someone else had their pull request merged into the same develop branch which had changes that may conflict with the file that you want to edit now.

Or even if you don't have a conflict it's still the best-case scenario to have the latest version of the application on your local computer.

You should do git pull origin develop.

Or, in full:

git checkout develop
git pull
git checkout -b feature/your-new-feature

What If Conflict Happens?
Let's simulate the scenario that we didn't pull down develop branch, so we will have a conflict.

Let's say we missed the latest changes from the develop branch and we do git checkout -b feature/user-country. Now let's add the country field, but look at the all fields: there is no about field which was added earlier by someone else, while we were working on our tasks.

app/Models/User.php:

// use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
 
class User extends Authenticatable
{
    // ...
    protected $fillable = [
        'name',
        'email',
        'password',
        'surname',
        'country', 
    ];
    // ...
}

Now let's push changes to the new branch and let's try to create a Pull Request.

git add .
git commit -m "User country"
git push --set-upstream origin feature/user-country

Now you see the message Can't automatically merge, which is a conflict.

conflict pr message

You can still create a Pull Request, but the reviewer will see that there is a conflict.

Typically what would happen then: a reviewer will contact you and ask to resolve the conflict, because they don't know what should be the latest version of the file.

conflict pr

Resolving Conflict
So currently we are in our feature/user-country branch. We do git checkout develop and pull down the latest changes with git pull origin develop.

Then we switch back to the feature/user-country branch by doing git checkout feature/user-country.

And then we merge the develop branch into the feature/user-country branch by doing git merge develop. And of course, we get the expected conflict message:

Auto-merging app/Models/User.php
CONFLICT (content): Merge conflict in app/Models/User.php
Automatic merge failed; fix conflicts and then commit the result.

Then we go into IDE, in my case PhpStorm, and it will exactly show the conflict like this:

phpstorm conflict

With those arrows and you have HEAD which is your current version of the code and then after equals are code from another branch, in this case, develop.

You need to resolve the conflict by leaving the version you need or merging two versions like this one, and removing those conflicts. So now we have both fields, country and about.

resolve conflict in phpstorm

Now that this conflict is resolved we need to commit those changes.

git add .
git commit -m "Resolve conflict"
git push

And after visiting this Pull Request you will see the latest commit with the message Resolve conflict and the approver may approve the pull request.

pr with resolved conflict

And if you go to Files changed you will that only one file with only this feature was changed.

correct file changes after conflict

So this is how you typically resolve the conflicts. But it's better to avoid them: just before creating a new feature branch always do git pull origin develop or whatever is your main branch. Maybe you don't use develop and then do git pull origin main.

That's it for this tutorial. This is my version of using git with branches. Different teams may have different branches and different philosophies/opinions.

If you want to video version on the same topic, you can check my 2-part YouTube videos:

[Git in Laravel. Part 1 - Branches: Main, Develop and Feature](https://www.youtube.com/watch?v=AmScEC-_72I)
[Git in Laravel. Part 2 - Conflicts and Better Pull Requests](https://www.youtube.com/watch?v=t020co_fROU)
