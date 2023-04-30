# Composer in Laravel: 9 Useful Features
Composer is a well-known tool to manage PHP project dependencies. But I'm pretty sure you're not using all of its features! In this tutorial, we'll show many less-known capabilities of Composer.

## 1. Versioning Syntax Options
Do you know the difference between the 1.3.*, ^1.3, and ~1.3 versions?

Composer uses quite a complex versioning system with a variety of operators. Let's look at the most common syntax options:
```
{
    "vendor/package": "1.3.2", // exactly 1.3.2
    "vendor/package": ">=1.3.2", // anything above or equal to 1.3.2
    "vendor/package": "<1.3.2", // anything below 1.3.2
    "vendor/package": "1.3.*", // >=1.3.0 <1.4.0
    "vendor/package": "1.*", // >=1.0.0 <2.0.0
    "vendor/package": "~1.3.2", // >=1.3.2 <1.4.0
    "vendor/package": "~1.3", // >=1.3.0 <2.0.0
    "vendor/package": "^1.3.2", // >=1.3.2 <2.0.0
    "vendor/package": "^0.3.2", // >=0.3.2 <0.4.0
    "vendor/package": "^1.1.0", // >=1.1.0 <2.0.0
}
```
There are more combinations supported by Composer, but you will mostly encounter these.

## 2. Check Outdated Packages
You might want to double-check which packages have newer versions, without actually running composer update. Run this:
```
composer outdated
```
And it will give you a list of packages with newer versions:

![image](https://user-images.githubusercontent.com/11309713/235369731-d855e4c1-d218-4128-a7c8-9fe2e00cec01.png)

In this list, you can see all the direct packages that have newer versions available. Some of them will automatically update with the composer update command, but some of them might require manual version change.

## 3. Bumping Versions
Last year Composer introduced a new command composer bump. Its main job is setting your application up with strict versions of packages. For example, we have Laravel 10 installed:
```
{
    "require-dev": {
        "laravel/framework": "^10.0"
    }
}
```
But that might break our application if we update it to a newer version. Even minor updates could have some impact. So if you prefer to use strict stable versions, you can run:
```
composer bump
```
Which will update your composer.json file to:
```
{
    "require-dev": {
        "laravel/framework": "10.5.0"
    }
}
```
Then, running composer update will not update any major or minor version of the package, it will stay stable at strictly 10.5.0.

You can read more about the bump command here or in Composer documentation.

## 4. Security Audit For Packages
Security is a big concern for any application, and using vulnerable packages might lead to a disaster.

Composer has a built-in command to check for any known vulnerabilities in your packages:
```
composer audit
```
Once you run this command, you'll see something like this:

![image](https://user-images.githubusercontent.com/11309713/235369793-063af72e-754d-4bf5-9f39-59a443eaba6d.png)

This means that in our case we have 1 package with a known vulnerability.

To prevent any unwanted issues, you can look at the package and see if you can suggest a fix with a Pull Request. Also, you may contact the creator of the package and wait for the next release.

## 5. Find Package Dependencies
You might run into a case where you won't be able to install a new package because of a conflict. A typical example is when you have a package with a specific version requirement, while there's already one installed with a different version.

To check what's using a package, you can use:
```
composer depends vendor/package
```
This will give you a list of all packages that use it:

![image](https://user-images.githubusercontent.com/11309713/235369812-ace1adf9-5f54-40b2-a450-4ae68094c77b.png)

Now you can check if you really need all of them, or if you have newer versions that would resolve a conflict.

## 6. Composer Scripts
If you ever looked at the composer.json file of default Laravel skeleton, you may have noticed a scripts section:
```
{
     "scripts": {
        "post-autoload-dump": [
            "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
            "@php artisan package:discover --ansi"
        ],
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ],
        "post-root-package-install": [
            "@php -r \"file_exists('.env') || copy('.env.example', '.env');\""
        ],
        "post-create-project-cmd": [
            "@php artisan key:generate --ansi"
        ]
    }
}
```
So what do those scripts do? Well, you can add a command that runs on specific actions.

For example, if you want to run a command after composer update you can add it to post-update-cmd:
```
{
    "scripts": {
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force",
            "@php artisan migrate"
        ]
    }
}
```
And now, every time you run composer update it will run migrations automatically.

You can even define your own scripts:
```
{
    "scripts": {
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ],
        "migrate": [
            "@php artisan migrate"
        ]
    }
}
```
This allows you to run composer migrate to run migrations. This is especially great if you have custom actions that need to be run to set up the local environment.

A few more ideas for scripts:

- Post-install: Create a .env file if it does not exist and generate the application key
- Post-update: Run tests automatically
These might not seem like much, but it automates a few steps!

## 7. Composer.lock File
While it might seem obvious what the composer.lock file does, it's important to understand the reasons behind its existence.

Let's see if you understand it correctly. Here's a quick explanation:

- composer.json file is used to define what packages you want to install
- composer.lock file is used to define what packages are already installed (with exact versions, and dependencies)
- This file should be pushed to the repository to ensure that your teammates and staging/production servers have exactly the same versions of packages. Otherwise, you might run into a case where your application works locally, but not on the production/teammate server.
- This file should NOT be modified manually. Otherwise, you might easily break your application.
All in all, the goal of this file is to "be on the same page" with the team and to ensure that your application works the same in all environments.

## 8. Composer Install Flags
The command composer install has many different flags available on it. Here's a quick list of the most useful ones:

- --dry-run - Shows what would be installed/updated, but does not install/update anything
- --no-dev - Does not install packages from the require-dev section. Great for production server!
- --ignore-platform-reqs - Ignores platform requirements (php & extensions). While it's dangerous and can cause bugs/issues, it might be useful (or even necessary) if you're running on a different platform than your application is designed for.
- --ignore-platform-req - Similar to the above, but ignores a specific platform requirement. For example, if you're running on PHP 8, but your application is designed for PHP 7.4, you can use --ignore-platform-req=php to ignore the PHP version requirement. Again, dangerous!.
- --optimize-autoloader - Optimizes autoloader for production, which makes it faster. Great for production server!
- --no-interaction - Does not ask for any input. Useful for CI/CD pipelines. And sometimes used in production as well.
To give an example, the typical production deployment command may include something like this:
```
composer install --no-dev --optimize-autoloader
```
As it will not install any dev packages (for example, PHPUnit or Laravel Breeze) and will optimize autoloader for production, making it faster.

## 9. Register Custom Files: Great for Helpers
Sometimes you might need to add some custom functionality to Laravel, like a global helper function that will be available in all your application.

You can do so by adding a file to app/helpers.php and then registering that file in composer.json:
```
{
    "autoload": {
        "files": [
            "app/helpers.php"
        ]
    }
}
```
This means that whatever you have in the app/helpers.php file will be available globally to be used anywhere. Great for some custom functionality!

That's all more or less known Composer features we wanted to list in this article. Would you add more?
