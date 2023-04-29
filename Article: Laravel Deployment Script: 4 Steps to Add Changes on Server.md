# Laravel Deployment Script: 4 Steps to Add Changes on Server
Deploying changes of your Laravel project to the server is not a simple process. I would separate that into 4 separate steps, or phases: code changes, dependencies, DB changes, and environment cleanup. Let's take a look at all of them, one by one.

So, you've pushed/merged the latest changes to the GitHub branch, you (or your auto deployment script robot) SSH into the server, and... do what?

## Phase 0. Putting the system down.
If we take into account those phases, the full deployment may take up to 5-10 seconds, or even more. What if during that time, someone is using your application and may make some database changes? Or, even worse, cause some conflicts between the old data/code and the new data/code?

To avoid that, you have two options:

1. Make the system inaccessible during deployment, with php artisan down.
2. Or, use one of the zero-downtime deployment tools (we'll talk about them later).
I typically advise performing artisan down at the very beginning of the deployment process. From then, all your app users would see the message "SERVICE UNAVAILABLE" with HTTP status 503, until you put the application back with php artisan up.

You may customize the default maintenance mode template by defining your own template at resources/views/errors/503.blade.php.

Now, as no one can access the app and make any risky interference, we can proceed with deployment.

## Phase 1. Pull the code changes.
This is probably the most straightforward part, just run git pull from the branch you're working with.
```
git pull origin main
```
For staging/testing servers, it may be
```
git pull origin develop
```
Of course, by that point, you have to have a git repository set up and configured earlier, including all the credentials and access, but this is outside of the scope of this article.

## Phase 2. Dependencies.
As a part of your code changes, you may have installed new packages, or update the existing ones, via composer.

The most typical way to deal with it is this sequence:

1. Locally you install the packages with composer require, or update versions with composer update
2. Those commands update the composer.lock file, which you push to the git repository
3. While deploying changes on the server, you just run composer install, and it will install all the changes registered in the composer.lock file
The rule is to almost never run composer update on the server, only composer install, otherwise you will have a much longer deployment process (composer update is slow), and also will have git conflicts because both sides changed the composer.lock.

A few useful flags may be added to the "composer install" command.
```
composer install --no-interaction --no-dev
```
The --no-interaction would prevent composer from asking you to confirm the command, in case you're running the automated script. Also, --no-dev would install only the packages from the "require" section of the composer.json and not the "require-dev".

## Phase 3. Database Changes.
What if you added any new migrations to your database schema?

As a part of your deployment script, you need to run this:
```
php artisan migrate
```
If you want to launch it automatically and avoid some prompt questions in the Terminal, you may add this flag:
```
php artisan migrate --force
```
What about potential new seeders, you would ask? There's an ongoing debate about how to deal with them.

Most people I've seen launch php artisan db:seed only once, at the very beginning of the project, and then forget about seeding.

Then, if they create a new DB table and need to fill it with the default data, they just do it inside the migration file itself:
```
return new class extends Migration
{
    public function up()
    {
        Schema::create('roles', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->timestamps();
        });
 
        Role::create(['name' => 'Administrator']);
        Role::create(['name' => 'Simple User']);
    }
```
Again, this is just one of the options, which you can fully automate. But if you are deploying manually, you can run some seeder in Terminal:
```
php artisan db:seed --class=RoleSeeder
```
## Phase 4. Environment Changes.
This phase can be actually optional. In here, you just restart what needs to be restarted, and clean what should be cleaned.

Here's a potential list of actions:
```
php artisan cache:clear
php artisan route:clear
php artisan view:clear
echo "" | sudo -S service php8.1-fpm reload
```
With these, you avoid potential problems related to caching or pieces of old code still being in action somewhere.

Also, you may restart the queues, so new jobs would use the new code.

## Phase Last. Go UP?
As mentioned before, don't forget to run php artisan up if you used php artisan down in the very beginning.

And now we can compile the full list of commands, which is our deployment script:
```
cd /home/some-subfolders/your-app-folder
php artisan down
git pull origin main
composer install --no-interaction --no-dev
php artisan migrate --force
php artisan cache:clear
php artisan route:clear
php artisan view:clear
echo "" | sudo -S service php8.1-fpm reload
php artisan up
```
See how many things may be happening?

That's why it is inconvenient to launch this sequence manually every time, so there are tools to launch this script with a click of a button, or even automatically when the GitHub branch receives the new changes.

Personally, I'm an old client of the official Laravel Forge, but its primary mission is actually provisioning the servers, deployment is just one of its functions.

Alternatively, there are tools specific for deployment - some free, some paid:

- Envoyer
- Ploi
- Deployer
And there are more general code deployment tools, not specific to Laravel or even PHP.

## Wait, So Zero-Downtime Deployment?
Yes, while talking about those deployment tools above, we need to get back to the option I had already mentioned.

Instead of doing php artisan down and disabling our application for 5-10 seconds or more, we can keep the app running, deploying the changes "in the background".

Personally, I use Envoyer for this - another official tool from the Laravel team. But again, the same previously mentioned tools allow zero-downtime deployment, too.

How does it work? In the case of Envoyer, it installs a new fresh Laravel application in a different "hidden" folder, runs all the commands from the deployment script, and then just changes the so-called symlink on the server, so the URL domain would point to the new folder. Since the symlink change happens almost instantly, that becomes the downtime of your app.

## Conclusion
So, these are typical actions that should happen during a deployment process, which can be executed manually from the Terminal, or automated with a script.

That said, each of those phases may contain different or extra commands, depending on your application, git options, server environment, and more.
