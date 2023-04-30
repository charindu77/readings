# Laravel Validation: Stock/Price Change with or without Livewire
What if your customer is filling in the order form, and meanwhile the product price has changed? Or, some product becomes out of stock? We need to re-validate the quantities/prices after the submit, right? In this article, I will show you two ways: regular Laravel and more UX-friendly "live validation" with Livewire.

![image](https://user-images.githubusercontent.com/11309713/235351291-7d572f01-8390-47f1-8f5d-70a3e44a048f.png)


Look at the validation messages above: this is exactly what we will be building. After the submit, the price of Product 1 changed in the DB, and also someone bought Product 2 so there are not enough items left.

It's a pretty complex validation, there's no ready-made Laravel validation rule in the official list. We need to create a Custom Validation Rule. But first, the initial project structure.

## Initial Structure
The database structure is simple - just one "products" table:

![image](https://user-images.githubusercontent.com/11309713/235351312-4379a416-cfb1-49fa-9392-047d7c007c2e.png)


I've created a simple demo project with the design based on our [Laravel Breeze Skeleton](https://github.com/LaravelDaily/Laravel-Breeze-Pages-Skeleton), two Routes, and one Controller.

#### routes/web.php:
```
Route::get('orders/create', [OrderController::class, 'create'])->name('orders.create');
Route::post('orders', [OrderController::class, 'store'])->name('orders.store');
```
####app/Http/Controllers/OrderController.php:
```
use App\Models\Product;
 
class OrderController extends Controller
{
    public function create()
    {
        $products = Product::all();
 
        return view('orders.create', compact('products'));
    }
}
```
The form in Laravel Blade looks like this:

####resources/views/orders/create.blade.php:
```
// ... here are the main layout tags - skipping them in this snippet
 
<x-validation-errors class="mb-4" :errors="$errors" />
 
<form action="{{ route('orders.store') }}" method="POST">
    @csrf
    <table>
        <thead>
        <tr>
            <th>Product</th>
            <th>Price</th>
            <th>Quantity</th>
        </tr>
        </thead>
 
        <tbody>
        @foreach($products as $product)
            <input type="hidden"
                name="prices[{{ $product->id }}]"
                value="{{ $product->price }}" />
 
            <tr>
                <td>{{ $product->name }}</td>
                <td>${{ number_format($product->price, 2) }}</td>
                <td>
                    <input type="number"
                        name="products[{{ $product->id }}]"
                        value="{{ old('products.' . $product->id, 0) }}" />
                </td>
            </tr>
        @endforeach
        </tbody>
    </table>
    <x-button>Place Order</x-button>
</form>
```
**Notice:** I intentionally deleted CSS classes and layout HTML tags here, to focus on the form functionality, you can find the original code with all classes in the Github repository.

As you can see, we pass two array parameters from the form:

- products[] as ID => quantity array
- prices[] as ID => price array in the hidden field
Now, the submission of the form goes to the store() method of the Controller, where we get to the main question:

#### app/Http/Controllers/OrderController.php:
```
use Illuminate\Http\Request;
 
class OrderController extends Controller
{
    // ...
 
    public function store(Request $request)
    {
        $request->validate([
            'products' => '????', // <- WHAT TO PUT HERE?
        ]);
 
        // ... Create the order and more
    }
}
```
## Custom Validation Rule
We need to validate two things for each of the products:

1. Has the price changed?
2. Do we have enough items in stock?
As I mentioned above, there's no available rule for this in Laravel, so we need to make our own.
```
php artisan make:rule ProductStockPriceRule --invokable
```
I will use the new "invokable" syntax of validation rules that appeared in Laravel 9, you can read about it in the official docs.

The initial generated structure of the rule is this:

####app/Rules/ProductStockPriceRule.php:
```
namespace App\Rules;
 
use Illuminate\Contracts\Validation\InvokableRule;
 
class Uppercase implements InvokableRule
{
    public function __invoke($attribute, $value, $fail)
    {
        // Validate here and call $fail('message') if validation fails
    }
}
```
Then, we apply that rule to our Controller:

#### app/Http/Controllers/OrderController.php:
```
use App\Rules\ProductStockPriceRule;
use Illuminate\Http\Request;
 
class OrderController extends Controller
{
    // ...
 
    public function store(Request $request)
    {
        $request->validate([
            'products' => new ProductStockPriceRule(),
        ]);
 
        // ... Create the order and more
    }
}
```
So, the products[] array will become the $value in our Custom Rule method. Now, let's fill it with the validation logic.

First, let's make a simple validation of checking if at least one product was chosen at all.
```
class ProductStockPriceRule implements InvokableRule
{
    public function __invoke($attribute, $value, $fail)
    {
        $products = array_filter($value);
        if (count($products) == 0) {
            $fail('Please select at least one product');
        }
 
        // If we don't call $fail, it means validation passed successfully
    }
}
```
Now, with the help of the array_filter() PHP method, we have $products as a key-value-based array of product IDs and quantities from the form, with filtered products where the quantity value is above 0.

Next, let's validate the product stock.

A typical performance mistake here would be to launch a foreach() loop, querying the DB for each product's quantity. Instead, let's query the database one time, get the results into a Collection, and then launch the foreach(), comparing the form quantities with the Collection data, instead of comparing with the DB.

#### app/Rules/ProductStockPriceRule.php:
```
use App\Models\Product;
 
class ProductStockPriceRule implements InvokableRule
{
    public function __invoke($attribute, $value, $fail)
    {
        $products = array_filter($value);
        if (count($products) == 0) {
            $fail('Please select at least one product');
        }
 
        $DBProducts = Product::find(array_keys($products))->keyBy('id');
 
        $errorText = '';
        foreach ($products as $productID => $quantity) {
            // Check stock
            if ($DBProducts[$productID]->stock_left < $quantity) {
                $errorText .= 'Sorry, we have only ' .
                    $DBProducts[$productID]->stock_left . ' of ' .
                    $DBProducts[$productID]->name . ' left in stock. ';
            }
        }
 
        if ($errorText != '') {
            $fail($errorText);
        }
    }
}
```
A few things to notice here.

- Did you know that Eloquent find() method accepts the array of IDs and then returns the Collection?
- We find Products in DB and use the Collection method keyBy('id') to be able to access the data like $DBProducts[$productID]
- We fill in the variable of $errorText with errors, because there may be multiple products out of stock
Next, validate the prices. To do that, we need to compare the DB prices to the ones from the "hidden" field. But the problem is that our validation rule is assigned to the "products" field, how do we additionally pass the "prices", too?

Invokable Validation Rules has this "trick" to access ALL the request:

- Implement the DataAwareRule interface
- Define the $data property
- Define the setData($data) method to assign it
- Then we will have $this->data['prices'] that we can use
```
use App\Models\Product;
use Illuminate\Contracts\Validation\DataAwareRule;
use Illuminate\Contracts\Validation\InvokableRule;
 
class ProductStockPriceRule implements InvokableRule, DataAwareRule
{
    protected array $data = [];
 
    public function setData($data)
    {
        $this->data = $data;
 
        return $this;
    }
 
    public function __invoke($attribute, $value, $fail)
    {
        // ...
 
        $errorText = '';
        foreach ($products as $productID => $quantity) {
            // Check stock...
 
            // Check price with $this->data['prices']
            if ($DBProducts[$productID]->price != $this->data['prices'][$productID]) {
                $errorText .= 'Sorry, the price of ' . $DBProducts[$productID]->name . ' has changed.
                    Old price: $' . number_format($this->data['prices'][$productID], 2) . ',
                    new price: $' . number_format($DBProducts[$productID]->price, 2) . '. ';
            }
        }
 
        if ($errorText != '') {
            $fail($errorText);
        }
    }
}
```
And that's it! We can now inform our customers, after they submit the form, that the stock/prices have changed.

![image](https://user-images.githubusercontent.com/11309713/235351506-31d1fad3-82df-458f-901f-c6f629f7e119.png)


Now, what if you want to inform them before the submission, in "live" mode?

Also, maybe you want to show the total order price and re-calculate it in "live" mode, as well?

For that, we will use Laravel Livewire. Let's dive into that second "advanced" part of this tutorial.

Showing Total Price with Livewire
First, before transforming our validation to live mode, I want to show you how Livewire works, in general.

So let's start by showing the current total price before submitting the form, re-calculating it on each change of any quantity field.

You can do it in various ways, possibly with JavaScript, but if you are a back-end developer, you may prefer writing NO JavaScript at all, so your choice could be Laravel Livewire. If you haven't worked with it at all, I have a separate 1.5-hour course Practical Laravel Livewire from Scratch, but in this tutorial, I will show you the step-by-step example for price calculation.

First, we install Livewire:
```
composer require livewire/livewire
```
Then, in our main layout Blade file, we need to include @livewireStyles in the <head> section and @livewireScripts before the </body>.

#### resources/views/layouts/app.blade.php:
```
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
    <head>
        // ...
        @vite(['resources/css/app.css', 'resources/js/app.js'])
        @livewireStyles
    </head>
    <body class="font-sans antialiased">
        <div class="min-h-screen bg-gray-100">
            // ...
        </div>
        @livewireScripts
    </body>
</html>
```
Then we will create a Livewire component that would replace our Create Order form.
```
php artisan make:livewire NewOrder
```
It will generate two files:

- app/Http/Livewire/NewOrder.php where will put the logic and variables
- resources/views/livewire/new-order.blade.php where we will cut-paste our Blade form, with extra wire:xxxxx "magic"
Now, let's take our resources/views/orders/create.blade.php and cut a big portion of it, and place it into a Livewire blade file.

#### resources/views/livewire/new-order.blade.php:
```
<form action="{{ route('orders.store') }}" method="POST">
    @csrf
 
    <x-validation-errors class="mb-4" :errors="$errors" />
    <table>
        <thead>
        <tr>
            <th>Product</th>
            <th>Price</th>
            <th>Quantity</th>
        </tr>
        </thead>
 
        <tbody>
        @foreach($products as $product)
            <input type="hidden"
                name="prices[{{ $product->id }}]"
                value="{{ $product->price }}" />
 
            <tr>
                <td>{{ $product->name }}</td>
                <td>${{ number_format($product->price, 2) }}</td>
                <td>
                    <input type="number"
                        name="products[{{ $product->id }}]"
                        value="{{ old('products.' . $product->id, 0) }}" />
                </td>
            </tr>
        @endforeach
        </tbody>
    </table>
    <x-button>Place Order</x-button>
</form>
```
And then we use our Livewire component in our main non-Livewire Blade file:

#### resources/views/order/create.blade.php:
```
<x-app-layout>
    // ... some more <div>'s for structure
    <div class="min-w-full align-middle">
        <x-validation-errors class="mb-4" :errors="$errors" />
 
        @livewire('new-order', ['products' => $products])
    </div>
</x-app-layout>
```
As you can see, we're passing the $products variable into our Livewire component, so for that we need to define it as a variable in the main Livewire component:

#### app/Http/Livewire/NewOrder.php:
```
namespace App\Http\Livewire;
 
use Illuminate\Support\Collection;
use Livewire\Component;
 
class NewOrder extends Component
{
    // Important to define the type here, like "Collection"
    public Collection $products;
 
    // This is generated automatically with "make:livewire"
    public function render()
    {
        return view('livewire.new-order');
    }
}
```
And that's it: we have re-created our form with Livewire, you can reload the page and see absolutely the same form!

So, this is our "introduction" to Livewire, without any dynamic behavior, for now. The main points are:

- You install and configure Livewire
- You create Livewire components which consist of a Component class and a Blade file
- You define the Component properties that would be accessible in the Livewire Blade file
- You call the component with @livewire() in non-Livewire regular Blade file
Now, let's calculate the total price.

In our Livewire Blade file, at the bottom, let's show the result variable:

#### resources/views/livewire/new-order.blade.php:
```
<table>
    <thead>
        // ...
    </thead>
 
    <tbody>
    @foreach($products as $product)
        // ...
    @endforeach
    </tbody>
    <tfoot>
        <tr>
            <th colspan="2"></th>
            <th class="px-6 py-4 text-left font-semibold">
                Total price: ${{ number_format($totalPrice, 2) }}
            </th>
        </tr>
    </tfoot>
</table>
```
Next, we need to define this $totalPrice as a property in the Livewire component.

#### app/Http/Livewire/NewOrder.php:
```
class NewOrder extends Component
{
    public Collection $products;
    public int $totalPrice = 0;
 
    public function render()
    {
        return view('livewire.new-order');
    }
}
```
The interesting part: if you define the properties in the Livewire component, you don't need to pass them to the Livewire Blade, they are available in the Blade automatically.

So, our price is hardcoded to 0 for now, and it shows below the table!

![image](https://user-images.githubusercontent.com/11309713/235351656-872b8ec9-62c0-4e62-a4dc-7571cdb5249d.png)

Now, let's actually use the Livewire power to make it dynamic.

For that, let's define a property $quantities which will contain the array of all the chosen quantities. In the Blade file, we change the old <input type="number" to this Livewire syntax:

Old code:

#### resources/views/livewire/new-order.blade.php:
                                                                                                                                                                                                                                                                                               ```                                                                                                                                                  
**<input type="number"
    name="products[{{ $product->id }}]"
    value="{{ old('products.' . $product->id, 0) }}" /> **

New code:

#### resources/views/livewire/new-order.blade.php:
```
<input type="number"
    wire:model="quantities.{{ $product->id }}"
    name="products[{{ $product->id }}]" />
```
The wire:model means that we're wiring this Blade input to the Livewire component property $quantities: whenever the input value changes, we will be able to access it in the Livewire component and make re-calculations.

Now we need to define those $quantities and assign the default values of 0 to them. For the initialization, Livewire has its own constructor-like method called mount():

#### app/Http/Livewire/NewOrder.php:
```
class NewOrder extends Component
{
    public Collection $products;
    public int $totalPrice = 0;
    public array $quantities = [];
 
    public function mount()
    {
        foreach ($this->products as $product) {
            $this->quantities[$product->id] = 0;
        }
    }
 
    public function render()
    {
        return view('livewire.new-order');
    }
}
```
Finally, we need to catch the updated quantity whenever it's changed. And this is where Livewire really shines.

All we need to do is define the method updatedProperty() in the Livewire component, in our case it would be called updatedQuantities(). In there, we can make the calculations and assign the value to $this->totalPrice which will be automatically re-rendered in the Blade!

#### app/Http/Livewire/NewOrder.php:
```
class NewOrder extends Component
{
    // ...
 
    public function updatedQuantities()
    {
        $this->totalPrice = 0;
        foreach ($this->quantities as $productId => $quantity) {
            $product = $this->products->find($productId);
            if ($product) {
                $this->totalPrice += $product->price * $quantity;
            }
        }
    }
}
```
We're searching for the product in the internal Collection (not in the database), and adding its price to the $totalPrice variable.

That's it, change some quantities and see the price changes immediately at the bottom!

![image](https://user-images.githubusercontent.com/11309713/235351899-e60f45f5-993f-41f0-9c64-a5bada705a1a.png)

## Validation: From Controller to Livewire Component
Now, as we've seen how Livewire works, we can transform the validation from after-submit to the dynamic Livewire component itself, so the full page wouldn't even refresh.

First, generally, with Livewire, you can enable any validation rules like this:

#### app/Http/Livewire/NewOrder.php:
```
use App\Rules\ProductStockPriceRule;
 
class NewOrder extends Component
{
    public function updatedQuantities()
    {
        $this->validate([
            'quantities' => new ProductStockPriceRule(),
        ]);
 
        // other functionality
    }
}
```
Instead of our custom rule, you could have any Laravel validation rule, like "required" or "array".

This validation will work with checking the quantity of stock, but we have a problem with price validation.

For the invokable rule, we implemented the DataAwareRule which will not work with Livewire, as it doesn't automatically pass the "prices" to the request.

For that, we will change the validation rule to specifically accept the prices in the Constructor.

#### app/Rules/ProductStockPriceRule.php:
```
namespace App\Rules;
 
use App\Models\Product;
use Illuminate\Contracts\Validation\InvokableRule;
 
class ProductStockPriceRule implements InvokableRule
{
    protected array $data = [];
 
    public function __construct(array $prices = [])
    {
        $this->data['prices'] = $prices;
    }
 
    public function __invoke($attribute, $value, $fail)
    {
        // all the other unchanged functionality as it was
    }
}
```
And that's it, now the validation will happen and the errors will appear as soon as we change the quantity!

The repository for this demo project can be found here: LaravelDaily/Laravel-Price-Stock-Validation-Demo
