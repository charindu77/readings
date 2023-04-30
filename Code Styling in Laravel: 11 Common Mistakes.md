# Code Styling in Laravel: 11 Common Mistakes
For fixing code styling mistakes, there's a great tool called Laravel Pint. In this article, I will list one of the most typical fixes it makes, with before/after examples.

## 1. Unused Imports
Unused imports are one of the most common code style issues in code. This can be caused by copying and pasting some code or simply by refactoring some classes. It usually looks like this:

![image](https://user-images.githubusercontent.com/11309713/235359802-125bab69-ee4a-4c28-a85d-f46d1d22d6e0.png)

While my editor marked them as unused - that might not be the case for your editor. That's why Pint has a rule about it:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'no_unused_imports' => true,
```
And once you run /vendor/bin/pint fix it will remove all unused imports:

![image](https://user-images.githubusercontent.com/11309713/235359867-302cfdc6-fcb2-4f34-96a1-7d4657f931b6.png)

![image](https://user-images.githubusercontent.com/11309713/235359877-a46e62cb-40d0-471c-bdac-bbf7e4e5d7a5.png)

## 2. Function Visibility
Functions in PHP should have their visibility defined even if it's public by default. This is a good practice, and it's also a rule in Pint:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'visibility_required' => [
    'elements' => ['method', 'property'],
],
```
Let's take a look at an example:
```
function getFullNameAttribute()
{
    return $this->first_name . ' ' . $this->last_name;
}
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235359913-63c34115-9f56-4825-985c-097fa22e7829.png)
```
public function getFullNameAttribute()
{
    return $this->first_name . ' ' . $this->last_name;
}
```
## 3. Useless Else Definition
If you have an if statement that returns something, you don't need to have an else statement. This is a good practice, and it's also possible to define this rule in Pint:

Open your pint.json file and add the following rule:

*pint.json*
```
{
  "rules": {
    "no_useless_else": true
  }
}
```
Let's take a look at an example:
```
if (auth()->user()->is_manager) {
    return view('manager.home');
} else {
    return view('admin.home');
}
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235359949-2500ff9a-356b-481c-a9ea-b64540111718.png)
```
if (auth()->user()->is_manager) {
    return view('manager.home');
}

return view('admin.home');
```
## 4. Function Declaration Spacing
Sometimes people tend to put a space between the function name and the opening parenthesis or parameters. This is not a good practice, and it's also a rule in Pint:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'function_declaration' => true,
```
Let's take a look at an example:
```
public function getReportsByProject ( $project,$raw)
{
    // ...
}
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235359987-16dde6b9-872a-4104-baa9-dd63d07f1a73.png)
```
public function getReportsByProject($project, $raw)
{
    // ...
}
```
In this example it might not be obvious but:

1. Removed space before ( so we have getReportsByProject(
2. Removed space before $project so it's ($project)
3. Added space before $user so it's ($project, $user)

## 5. Single Line After Imports
You might have noticed that sometimes in your code you have various empty lines after imports. This is not a good practice, and it's also a rule in Pint:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
``
'single_line_after_imports' => true,
``
Let's take a look at an example:
```
namespace App\Http\Controllers\Admin;
 
use App\Http\Controllers\Controller;
use App\Http\Requests\MassDestroyProjectRequest;
use App\Http\Requests\StoreProjectRequest;
use App\Http\Requests\UpdateProjectRequest;
use App\Models\Client;
use App\Models\Project;
use App\Models\ProjectStatus;
use Gate;
use Symfony\Component\HttpFoundation\Response;
 
 
 
 
class ProjectController extends Controller {
    // ...
}
```
As you can see there are 3 empty lines after the imports. Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360073-d0f42171-d2e1-4293-ad34-0093b6b87a6a.png)
```
namespace App\Http\Controllers\Admin;
 
use App\Http\Controllers\Controller;
use App\Http\Requests\MassDestroyProjectRequest;
use App\Http\Requests\StoreProjectRequest;
use App\Http\Requests\UpdateProjectRequest;
use App\Models\Client;
use App\Models\Project;
use App\Models\ProjectStatus;
use Gate;
use Symfony\Component\HttpFoundation\Response;
 
class ProjectController extends Controller
{
    // ...
}
```
## 6. Combining Isset Function
A great example of a less-known feature in your code - you can combine multiple isset() checks into one. And Pint can do that change for you if you ever forget it!

Open your pint.json file and add the following rule:

*pint.json*
```
{
  "rules": {
    "combine_consecutive_issets": true
  }
}
```
Let's look at the example:
```
if(isset($projectID) && isset($clientID)){
    // ...
}
```
While it works perfectly fine - running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360109-4b40a154-d783-47ec-b899-9e987c5cc865.png)
```
if (isset($projectID, $clientID)) {
    // ...
}
```
Looks much cleaner!

## 7. Array Syntax
Since php have an old definition of arrays by typing array() you might still encounter it in your code. That said - the standard right now is []. That's where Pint helps to refactor this too

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'array_syntax' => ['syntax' => 'short'],
```
Let's look at the example:
```
// ...
$entries[$currency][$date] = array(
    'income' => 0,
    'expenses' => 0,
    'fees' => 0,
    'total' => 0,
);
// ...
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360153-bb2c97d6-71bf-4f87-8aa8-2a9e09ceb8cf.png)
```
$entries[$currency][$date] = [
    'income' => 0,
    'expenses' => 0,
    'fees' => 0,
    'total' => 0,
];
```
## 8. Spacing Around String Concatenation
When combining multiple variables or strings with variables we are using a . to do that. But there's a Pint rule that defines how many spaces we have to have:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'concat_space' => [
    'spacing' => 'none',
],
```
In the case of Laravel preset in Pint that's none. Here's what that looks like:
```
echo 'First name: ' . $user->first_name . ' and Last name: ' . $user->last_name;
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360210-eeaee908-8515-4dd4-9c2d-3cb5a47b4683.png)
```
echo 'First name: '.$user->first_name.' and Last name: '.$user->last_name;
```
As you can see - there are no more spaces around the ..

## 9. Using Double Quotes
This is quite old issue where people tend to use " instead of '. Pint can take care of this and fix it:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'single_quote' => true,
```
Let's look at an example code:
```
$array = [
    "project1" => [],
    "project2" => []
];

return $array["project1"];
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360257-86f4207f-121a-4163-b81a-48edc3f0a6ec.png)

```
$array = [
    'project1' => [],
    'project2' => []
];
 
return $array['project1'];
```
## 10. Blank Lines After Namespace
When declaring a namespace at the top of the file - it's quite common to leave an empty line of space. Pint takes care of this and fixes it:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'blank_line_after_namespace' => true,
```
```
<?php
 
namespace App\Http\Controllers\Admin;
use App\Http\Controllers\Controller;
use App\Http\Requests\MassDestroyProjectRequest;
use App\Http\Requests\StoreProjectRequest;
use App\Http\Requests\UpdateProjectRequest;
use App\Models\Client;
use App\Models\Project;
use App\Models\ProjectStatus;
use Gate;
use Symfony\Component\HttpFoundation\Response;
 
class ProjectController extends Controller {
    // ...
}
```
Running /vendor/bin/pint fix will change it to:

![image](https://user-images.githubusercontent.com/11309713/235360323-3d78be27-af53-4f51-bac6-3afcb1825a50.png)
```
<?php
 
namespace App\Http\Controllers\Admin;
 
use App\Http\Controllers\Controller;
use App\Http\Requests\MassDestroyProjectRequest;
use App\Http\Requests\StoreProjectRequest;
use App\Http\Requests\UpdateProjectRequest;
use App\Models\Client;
use App\Models\Project;
use App\Models\ProjectStatus;
use Gate;
use Symfony\Component\HttpFoundation\Response;
 
class ProjectController extends Controller {
    // ...
}
```
It gives you a much cleaner separation of the namespace from all the imports.

## 11. Class Definition Structure
A lot of people like to leave a bracket on the same line as the Class definition but Laravel coding style enforced by Pint recommends you move it to the next line:

[https://github.com/laravel/pint/blob/main/resources/presets/laravel.php](https://github.com/laravel/pint/blob/main/resources/presets/laravel.php)
```
'curly_braces_position' => [
    'classes_opening_brace' => 'next_line_unless_newline_at_signature_end',
],
```
Let's look at an example:
```
class ProjectController extends Controller {
    // ...
}



class ProjectController extends Controller
{
    // ...
}
```
