# Laravel Api Auth with Vue and Sanctum: All You Need To Know
One of the ways to become a full-stack developer is to adapt Laravel + Vue.js pair. And part of that is authentication. In this tutorial, we will explore how to use Laravel, Vue, and Laravel Sanctum together to build an API authentication, in two ways:

- In two-in-one Laravel + Vue SPA
- Or, as separate Vue + API projects
Are you ready? Let's dive in!

## Project 1. Laravel SPA: Breeze Vue Example
To have a quick head start, Laravel Breeze starter kit provides a minimal, simple implementation of all Laravel's authentication features. Laravel Breeze also offers Vue scaffolding via an Inertia frontend implementation.

First, create a new Laravel project and install Laravel Breeze:
```
composer require laravel/breeze --dev
```
After that execute breeze:install Artisan command with Vue stack and all auth scaffolding will be installed, you should also compile your application's frontend assets:
```
php artisan breeze:install vue
 ```
 ```
php artisan migrate
npm install
npm run dev
```
Now you have a full working Single Page Application (SPA). Authentication controllers are placed in the app/Http/Controllers/Auth folder. Let's lookup at the app/Http/Controllers/Auth/AuthenticatedSessionController.php file's store method:
```
public function store(LoginRequest $request): RedirectResponse
{
    $request->authenticate();
 
    $request->session()->regenerate();
 
    return redirect()->intended(RouteServiceProvider::HOME);
}
```
This method is called when you log in to your application. As we can see there are no references to tokens. That's right, Vue and Inertia scaffolding uses the laravel_session cookie for authenticated sessions and is handled automatically, so no additional implementation is needed.

Let's move forward with the current setup and create a "protected" demo component that is accessible only to authorized users and displays the currently logged-in user's id and name.

First create a new controller app/Http/Controllers/DemoController.php with the following command.
```
php artisan make:controller DemoController
```
And this is the content of the file:
```
<?php
 
namespace App\Http\Controllers;
 
use Illuminate\Http\Request;
use Inertia\Inertia;
 
class DemoController extends Controller
{
    public function index()
    {
        return Inertia::render('Demo/Index');
    }
}
```
Add a route to routes/web.php for this controller:
```
Route::get('/demo', [DemoController::class, 'index'])->name('demo');
```
It should be under auth middleware along profile routes:
```
use App\Http\Controllers\DemoController;
 
// ...
 
Route::middleware('auth')->group(function () {
    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');
    Route::get('/demo', [DemoController::class, 'index'])->name('demo');
});
```
Add a link into the menu for the new demo route in the resources/js/Layouts/AuthenticatedLayout.vue file:
```
<NavLink :href="route('demo')" :active="route().current('demo')">
    Demo
</NavLink>
```
It can be right after the Dashboard:
```
<!-- Navigation Links -->
<div class="hidden space-x-8 sm:-my-px sm:ml-10 sm:flex">
    <NavLink :href="route('dashboard')" :active="route().current('dashboard')">
        Dashboard
    </NavLink>
    <NavLink :href="route('demo')" :active="route().current('demo')">
        Demo
    </NavLink>
</div>
```
And finally Vue component resources/js/Pages/Demo/Index.vue itself with the following content:
```
<script setup>
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';
import { Head, usePage } from '@inertiajs/vue3';
 
const user = usePage().props.auth.user
</script>
 
<template>
    <Head title="Demo" />
 
    <AuthenticatedLayout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Demo
            </h2>
        </template>
 
        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                    <div class="p-6 text-gray-900">My protected content</div>
                    <div class="p-6 text-gray-900">
                        <div>Id: {{ user.id }}</div>
                        <div>Name: {{ user.name }}</div>
                    </div>
                </div>
            </div>
        </div>
    </AuthenticatedLayout>
</template>
```
To display currently logged-in user data using blade files usually, we use auth()->user(). When using Vue and Inertia equivalent to displaying user data is the usePage() function which gives us access to globally shared data with the session.

If we check the app/Http/Kernel.php file we have two new middlewares added:
```
\App\Http\Middleware\HandleInertiaRequests::class,
\Illuminate\Http\Middleware\AddLinkHeadersForPreloadedAssets::class,

Interesting is the first middleware. We look up at contents in the app/Http/Middleware/HandleInertiaRequests.php file.

public function share(Request $request): array
{
    return array_merge(parent::share($request), [
        'auth' => [
            'user' => $request->user(),
        ],
        'ziggy' => function () use ($request) {
            return array_merge((new Ziggy)->toArray(), [
                'location' => $request->url(),
            ]);
        },
    ]);
}
```
This is where globally shared data for currently logged-in users is added under the auth key, which corresponds to the usePage().props.auth.user line in a Vue component.

So far so great, usually if the front end is the only consumer of the backend there's no need to use an API.

## Sanctum and SPA Authentication
Suppose you intend to share identical data with both your application and external third-party services that use your API or have any other good reason. In such a scenario, using an API would be preferable. Employing a single "source of truth" eliminates the need to maintain multiple separate locations for the same information.

Laravel Sanctum offers a simple way to authenticate SPA that needs to communicate with Laravel API. It can authenticate using cookies from the Laravel session if you are currently authenticated (stateful) or use API tokens (stateless).

Endpoint to retrieve authenticated user's data is already defined and comes with the default Laravel installation. It is located in the routes/api.php file. As we can see it is protected by auth:sanctum middleware.
```
Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```
1. Update the resources/js/Pages/Demo/Index.vue component with the following content:
```
<script setup>
import { onMounted, reactive } from 'vue';
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';
import { Head } from '@inertiajs/vue3';
 
const state = reactive({
    user: {}
})
 
function fetchUser() {
    return axios.get('api/user')
        .then(response => state.user = response.data);
}
 
onMounted(fetchUser)
 
</script>
 
<template>
    <Head title="Demo" />
 
    <AuthenticatedLayout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Demo
            </h2>
        </template>
 
        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                    <div class="p-6 text-gray-900">My protected content</div>
                    <div class="p-6 text-gray-900">
                        <div>Id: {{ state.user.id }}</div>
                        <div>Name: {{ state.user.name }}</div>
                    </div>
                </div>
            </div>
        </div>
    </AuthenticatedLayout>
</template>
```
Now when the component is loaded it will try to fetch data from api/user. But there's a problem, the request will get rejected with 401 Unauthorized status.

2. You should add Sanctum's middleware to your api middleware group within your app/Http/Kernel.php file. This middleware is responsible for ensuring that incoming requests from your SPA can authenticate using Laravel's session cookies, while still allowing requests from third parties or mobile applications to authenticate using API tokens:
```
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    \Illuminate\Routing\Middleware\ThrottleRequests::class . ':api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```
Sanctum will only attempt to authenticate using cookies when the incoming request originates from your own SPA front end.

When Sanctum examines an incoming HTTP request, it will first check for an authentication cookie and, if none is present, Sanctum will then examine the Authorization header for a valid API token. We will cover API tokens in the next chapter.

*In order to authenticate, your SPA and API must share the same top-level domain. However, they may be placed on different subdomains*

3. Setting up environment variables

In order to get Sanctum to authenticate our requests we need to specify which domains of our application should be treated as stateful. This can be done by specifying SANCTUM_STATEFUL_DOMAINS in our .env file.

*Requests from the following domains/hosts will receive stateful API authentication cookies. Typically, these should include your local and production domains which access your API via a frontend SPA.*

If you have configured the local domain and API is deployed under the same domain it is sufficient to only specify the correct APP_URL in your .env file, for example:
```
APP_URL=http://myproject.test
```
Sanctum will try to resolve the SANCTUM_STATEFUL_DOMAINS value by inheriting the domain value from APP_URL if possible. In case your APP_URL is not defined or doesn't match the URL in the browser SANCTUM_STATEFUL_DOMAINS should be defined explicitly:
```
SANCTUM_STATEFUL_DOMAINS=myproject.test
```
Sometimes you might run the application using the php artisan serve command, and then API authentication wouldn't work.

In such case if you are accessing your application via a URL that includes a port (127.0.0.1:8000) like using mentioned php artisan serve command, you should define the SANCTUM_STATEFUL_DOMAINS environment variable and ensure that you include the port number with the domain:
```
SANCTUM_STATEFUL_DOMAINS=127.0.0.1:8000
```

## Project 2. Vue Client + Laravel API
### Sanctum and API Token Authentication
Sanctum allows you to issue API tokens that may be used to authenticate API requests to your application. In this chapter, we will create a separate front-end client to consume API powered by Laravel.

Now for the API server, we can reuse the same application from chapter one because it is already up and running, but if needed, you can create a new Laravel project for this purpose.

1. To begin issuing tokens for users, your User model should use the Laravel\Sanctum\HasApiTokens trait:
```
use Laravel\Sanctum\HasApiTokens;
 
class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```
It is very likely that you already have the HasApiTokens trait present in your User model if the project was created recently.

2. To issue a token, we use the createToken method. It returns a Laravel\Sanctum\NewAccessToken instance, but you may access the plain-text value of the token using the plainTextToken property of the NewAccessToken instance.
To allow users to "log in" and "logout" using the API you need to add corresponding routes to your routes/api.php file:
```
use App\Models\User;
use Illuminate\Support\Facades\Auth;
 
Route::post('/login', function (Request $request) {
    if (! Auth::attempt($request->only('email', 'password'))) {
        return response(['message' => __('auth.failed')], 422);
    }
 
    $token = auth()->user()->createToken('client');
 
    return ['token' => $token->plainTextToken];
});
 
Route::middleware('auth:sanctum')->post('/logout', function (Request $request) {
    $request->user()->currentAccessToken()->delete();
 
    return response()->noContent();
});
```
When a user accesses the /login route, we verify the provided credentials. If they are valid, we generate a new token and return it to the user. Otherwise, we send a message indicating authentication failure with a status code of 422 Unprocessable Entity.

If a user initiates a logout request, the token used to authenticate the request will be deleted. It is important to note that the /logout route is safeguarded by the auth:sanctum middleware, which ensures that only authenticated users can request the removal of their own tokens.

The logic for these routes can be put into controllers, but for clarity, in this tutorial, we will leave it as is.

### Vue Client
In this section, we will introduce how to set up a Vue Single Page Application. The created project will be using a build setup based on Vite and will consume API from a separate Laravel project.

Make sure you have an up-to-date version of Node.js installed, then run the following command in your command line:
```
npm init vue@latest
```
We have selected the following options:
```
✔ Project name: … myproject
✔ Add TypeScript? … No
✔ Add JSX Support? … No
✔ Add Vue Router for Single Page Application development? … Yes
✔ Add Pinia for state management? … No
✔ Add Vitest for Unit Testing? … No
✔ Add an End-to-End Testing Solution? › No
✔ Add ESLint for code quality? … No
```
Note that we chose to install Vue Router for routing. This is an important step not to miss.
```
✔ Add Vue Router for Single Page Application development? … Yes
```
Once the project is created, follow the instructions to install dependencies and start the dev server:
```
cd myproject
npm install
npm run dev
```
When the server starts you will be prompted that the server is ready and the URL to access it will be shown:
```
VITE v4.1.4  ready in 244 ms
 
➜  Local:   http://localhost:5173/
➜  Network: use --host to expose
➜  press h to show help
```
1. First, we need to install the Axios library for requests to API:
```
npm install axios --save
```
Then add the following content to the src/main.js file:
```
import axios from 'axios'
 
window.axios = axios
axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest'
axios.defaults.baseURL = 'http://<YOUR-LARAVEL-API-SERVER>/api'
 
if (localStorage.getItem('token')) {
  axios.defaults.headers.common['Authorization'] = `Bearer ${localStorage.getItem('token')}`
}
 
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      axios.defaults.headers.common['Authorization'] = 'Bearer'
      router.push({ name: 'login' })
    }
 
    return Promise.reject(error);
  }
);
```
We set the X-Requested-With header to tell the server it is an XHR request, and it serves an additional purpose so the server must consent to CORS policies.

The convenience option is axios.defaults.baseURL = "http://parkingapi.test/api/v1"; so we can omit full URLs in our requests and just type in the relative path of the server's API endpoint.

We are going to store the token in the browser's localStorage with a key token. When the client is loaded it will immediately try to retrieve the token from localStorage and set the Authorization header for future axios requests. This is done with the following code section:
```
if (localStorage.getItem('token')) {
  axios.defaults.headers.common['Authorization'] = `Bearer ${localStorage.getItem('token')}`
}
```
If any request to the backend fails due to an expired or invalid token with the 401 Unauthenticated status we need to set up an Axios interceptor. The concept of interceptor is basically the same as working with Laravel middleware.

Interceptor clears the token from the storage and Axios header. As a result, the user will be redirected to the login page. Implementation can be observed in the following snippet:
```
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      axios.defaults.headers.common['Authorization'] = 'Bearer'
      router.push({ name: 'login' })
    }
 
    return Promise.reject(error);
  }
);
```
The full content of the src/main.js file now should look like that:
```
import { createApp } from 'vue'
import axios from 'axios'
import App from './App.vue'
import router from './router'
 
import './assets/main.css'
 
window.axios = axios
axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest'
axios.defaults.baseURL = 'http://<YOUR-LARAVEL-API-SERVER>/api'
 
if (localStorage.getItem('token')) {
  axios.defaults.headers.common['Authorization'] = `Bearer ${localStorage.getItem('token')}`
}
 
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      axios.defaults.headers.common['Authorization'] = 'Bearer'
      router.push({ name: 'login' })
    }
 
    return Promise.reject(error);
  }
);
 
const app = createApp(App)
 
app.use(router)
 
app.mount('#app')
```
2. Create a new src/views/LoginView.vue component with the following content:
```
<script setup>
import { reactive, ref } from 'vue'
import { useRouter } from 'vue-router'
 
const router = useRouter()
 
const form = reactive({
  email: '',
  password: '',
})
 
const message = ref()
 
function submit() {
  message.value = ''
 
  axios.post('login', form)
    .then(response => {
      localStorage.setItem('token', response.data.token)
      axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`
      router.push({ name: 'user' })
    })
    .catch(error => {
      if (error.response.status === 422) {
        message.value = error.response.data.message
      }
    })
    .finally(() => form.password = '')
}
</script>
 
<template>
  <div>
    <p v-if="message" class="error">{{ message }}</p>
    <form @submit.prevent="submit" class="login">
      <div class="form-group">
        <label>Email</label>
        <input v-model="form.email" type="text" class="form-input">
      </div>
      <div class="form-group">
        <label>Password</label>
        <input v-model="form.password" type="password" class="form-input">
      </div>
      <div class="form-group">
        <button type="submit" class="form-input">
          Login
        </button>
      </div>
    </form>
  </div>
</template>
 
<style>
.login {
  font-size: 1.2em;
  display: flex;
  flex-direction: column;
  gap: 1em;
}
.form-group {
  display: flex;
  flex-direction: column;
}
.form-input {
  padding: 0.5em;
  font-size: 1em;
}
.error {
  color:red;
  font-size: 1em;
}
</style>
```
Our login form has email and password fields with their values bound to the form reactive state. message is used to display validation errors for demonstration purposes. When the form is submitted the submit method will be invoked and the token will be saved in the client:
```
function submit() {
  message.value = ''
 
  axios.post('login', form)
    .then(response => {
      localStorage.setItem('token', response.data.token)
      axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`
      router.push({ name: 'user' })
    })
    .catch(error => {
      if (error.response.status === 422) {
        message.value = error.response.data.message
      }
    })
    .finally(() => form.password = '')
}
```
Remember we had set axios.defaults.baseURL = 'http://<YOUR-LARAVEL-API-SERVER>/api' in the src/main.js file. So the following call of axios.post('login', form) is equivalent to axios.post('http://<YOUR-LARAVEL-API-SERVER>/api/login', form). The second argument is the form data we are submitting.

then() section will be executed if the authentication attempt was successful. The token will be stored on the client and the Axios header is updated and the user redirected to the UserView component using the named route user.

The catch() section is executed if the request to authenticate has been denied due to invalid credentials and the message value will be updated and displayed on the client.

The finally() section is executed always when the request is resolved and will clear the password field in the form.

3. Create a new src/views/UserView.vue component:
 ```
  <script setup>
import { onMounted, reactive } from 'vue'
import { useRouter } from 'vue-router'
 
const router = useRouter()
 
const state = reactive({
  user: {}
})
 
function getUser() {
  axios.get('user').then(response => {
      state.user = response.data
  })
}
 
function logout() {
  axios.post('logout').finally(() => {
    localStorage.removeItem('token')
    axios.defaults.headers.common['Authorization'] = 'Bearer'
    router.push({ name: 'login' })
  })
}
 
onMounted(getUser)
</script>
<template>
  <div>
    <div>ID: {{ state.user.id }}</div>
    <div>Email: {{ state.user.email }}</div>
    <div>
      <button @click="logout" type="button">Logout</button>
    </div>
  </div>
</template>
```
When the UserView component is mounted it will automatically call the getUser() method using the onMounted() hook and will fetch data from the /api/user endpoint setting response data to state.user.
```
function logout() {
  axios.post('logout').finally(() => {
    localStorage.removeItem('token')
    axios.defaults.headers.common['Authorization'] = 'Bearer'
    router.push({ name: 'login' })
  })
}
```
The logout method sends a request to the server to delete the token in use so it will become invalid and requests will no longer be valid. After sending the request client ignores what the response server gave and always "forces" logout by deleting the token and clearing Axios Authentication header.

4. Finally to "glue" all the pieces define routes for LoginView and UserView in the src/router/index.js file:
```
import LoginView from '../views/LoginView.vue'
import UserView from '../views/UserView.vue'

{
  path: '/login',
  name: 'login',
  component: LoginView
},
{
  path: '/user',
  name: 'user',
  component: UserView
},
```
The full contents of the src/router/index.js file look like this.
```
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'
import LoginView from '../views/LoginView.vue'
import UserView from '../views/UserView.vue'
 
const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView
    },
    {
      path: '/login',
      name: 'login',
      component: LoginView
    },
    {
      path: '/user',
      name: 'user',
      component: UserView
    },
    {
      path: '/about',
      name: 'about',
      // route level code-splitting
      // this generates a separate chunk (About.[hash].js) for this route
      // which is lazy-loaded when the route is visited.
      component: () => import('../views/AboutView.vue')
    }
  ]
})
 
export default router
```
You now have at your disposal all the essential examples for utilizing Sanctum authentication to consume Laravel API, whether it is for a consolidated application that uses cookies for the stateful session, or for two entirely separate repositories: one for the API and one for the frontend.
