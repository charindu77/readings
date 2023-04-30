# Laravel Testing: Mocking/Faking External 3rd Party APIs
Laravel PHPUnit Mocking

One of the most common questions about automated testing in Laravel is how to write tests for the usage of some external API. There are three ways to do that, and I will show all of those in this article.

This is a text-based excerpt of one chapter in my course Advanced Laravel Testing

We will talk about:

- Faking HTTP requests
- Mocking Classes
- Using External Sandboxes
As an example, let's imagine you use an external service to get the information by IP address, with Laravel HTTP Client.
```
private static function getIpData(string $ip): array
{
    $token = config('location.ipdata.token', '');
 
    $url = "https://api.ipdata.co/{$ip}?api-key=".$token;
 
    return Http::get($url)->throw()->json();
}
```
How would you test if it works?

The code above is from a real open-source project Monica. Let's take a look at our options.

## Option 1. Faking HTTP Requests with Http::fake()
The project authors fake the HTTP requests for external services.

They also use a thing called Fixtures, which are just pre-prepared sets of data. So, if we take a look at tests/Fixtures, there are prepared sets of data in JSON format. Ipdata.json contains an example response from an external service about IP. So, the IP address should contain this example information:
```
{
    "ip": "12.34.56.78",
    "is_eu": true,
    "city": "Paris",
    "region": "\u00cele-de-France",
    "region_code": "IDF",
    "country_name": "France",
    "country_code": "FR",
    "continent_name": "Europe",
    "continent_code": "EU",
    "latitude": 0,
    "longitude": 0,
    "postal": "75000",
    "calling_code": "33",
    "flag": "https://ipdata.co/flags/fr.png",
    "emoji_flag": "\ud83c\uddeb\ud83c\uddf7",
    "emoji_unicode": "U+1F1EB U+1F1F7",
    "languages": [
        {
            "name": "French",
            "native": "Fran\u00e7ais"
        }
    ],
    "currency": {
        "name": "Euro",
        "code": "EUR",
        "symbol": "\u20ac",
        "native": "\u20ac",
        "plural": "euros"
    },
    "time_zone": {
        "name": "Europe/Paris",
        "abbr": "CET",
        "offset": "+0100",
        "is_dst": false,
        "current_time": "2021-11-01T00:00:00+01:00"
    },
    "threat": {
        "is_tor": false,
        "is_proxy": false,
        "is_anonymous": false,
        "is_known_attacker": false,
        "is_known_abuser": false,
        "is_threat": false,
        "is_bogon": false
    },
    "count": "0"
}
```
Now, how those example JSON responses are used in tests? This ipdata.json file usage can be found in the tests/Unit/Helpers/RequestHelperTest.php method get_infos_from_ip():
```
class RequestHelperTest extends TestCase
{
    /** @test */
    public function get_infos_from_ip()
    {
        config(['location.ipdata.token' => 'test']);
 
        $body = file_get_contents(base_path('tests/Fixtures/Helpers/ipdata.json'));
        Http::fake([
            'https://api.ipdata.co/*' => Http::response($body, 200),
        ]);
 
        $this->assertEquals(
            [
                'country' => 'FR',
                'currency' => 'EUR',
                'timezone' => 'Europe/Paris',
            ],
            RequestHelper::infos('test')
        );
    }
}
```
Now, what does this test do?

First, it overwrites the config, so we don't expose and don't use the live token of that service ipdata.co.

Then, we use Http::fake(). So you can fake the request to any URL, here https://api.ipdata.co/* they use asterisks so that whatever URL with api.ipdata.co would be faked with the response. That response comes from the ipdata.json you saw earlier.

So we fake the API request to api.ipdata.co so it would not make the real HTTP request, but instead, it would return data from the ipdata.json file.

And then, finally, we assert that RequestHelper::infos() returns an array of:
```
[
    'country' => 'FR',
    'currency' => 'EUR',
    'timezone' => 'Europe/Paris',
],
```
What is inside of the RequestHelper::infos()?
```
public static function infos($ip)
{
    $ip = $ip ?? static::ip();
 
    if (config('location.ipstack_apikey') != null) {
        $ipstack = new Ipstack(config('location.ipstack_apikey'));
        $position = $ipstack->get($ip, true);
 
        if ($position !== null && Arr::get($position, 'country_code')) {
            return [
                'country' => Arr::get($position, 'country_code'),
                'currency' => Arr::get($position, 'currency.code'),
                'timezone' => Arr::get($position, 'time_zone.id'),
            ];
        }
    }
 
    if (config('location.ipdata.token') != null) {
        try {
            $position = static::getIpData($ip);
 
            return [
                'country' => Arr::get($position, 'country_code'),
                'currency' => Arr::get($position, 'currency.code'),
                'timezone' => Arr::get($position, 'time_zone.name'),
            ];
        } catch (\Exception $e) {
            // skip
        }
    }
 
    return [
        'country' => static::country($ip),
        'currency' => null,
        'timezone' => null,
    ];
}
```
First, it checks which external service to use - Monica project is configured to potentially use different external APIs for this. In this case, we use the ipdata service, so the second block is executed with static::getIpData($ip) inside.

In other words, the sequence is this:

- There is some service/helper class method that calls the external API
- We override the result of that API call with our static data and Http::fake()
- In the test, we call that service/helper class and assert there are no errors

## Option 2. Mocking Classes
The word "mocking" is actually a more sophisticated word for faking the object, like a PHP class. Now, let's take a look at how it is implemented in Laravel.

Let's imagine we have a Product model which has youtube_id and youtube_thumbnail columns, where you will store YouTube video ID and its thumbnail, taken from the external Youtube API.

To get that thumbnail, you create a separate Service class called YoutubeService with the method getThumbnailByID(). Then, in the Controller when creating the record, if the youtube_id field is set, then youtube_thumbnail will be auto-set from YoutubeService.
```
class ProductController extends Controller
{
    public function store(StoreProductRequest $request, YouTubeService $youTubeService)
    {
        $productData = $request->validated();
 
        if ($request->youtube_id) {
            $prooductData['youtube_thumbnail'] = $youTubeService->getThumbnailByID($request->youtube_id);
        }
 
        $product = Product::create($productData);
 
        return redirect()->route('products.index');
    }
}
```
Now, what will that YoutubeService look like:

#### app/Services/YoutubeService.php:
```
class YouTubeService
{
    public function getThumbnailByID(string $youtubeID)
    {
        $response = Http::asJson()
            ->baseUrl('https://youtube.googleapis.com/youtube/v3/')
            ->get('videos', [
                'part' => 'snippet',
                'id' => $youtubeID,
                'key' => config('services.youtube.key'),
            ])->collect('items');
 
        return $response[0]['snippet']['thumbnails']['default']['url'];
    }
}
```
This service may be more complex, but the main thing is that it makes the API request to collect the thumbnail. The tricky part here is that we need to provide the API key: YouTube is not a public API, and it won't give us the data without the key.

We can test this behavior in PHPUnit from two angles:

- First, test the actual service class, so that it works correctly
- Or, you may test the actual HTTP request of your project and skip the YouTube service by faking its data.
In most cases, if you use the API correctly following the official docs, you can be pretty sure that it will return the correct result. There's quite a little chance of Youtube API goes down, right?

So, in your tests, you should better test your application behavior, assuming that the external API call will be successful. Let's see an example of Mocking.
```
$this->mock(YouTubeService::class)
    ->shouldReceive('getThumbnailByID')
    ->with('5XywKLjCD3g')
    ->once()
    ->andReturn('https://i.ytimg.com/vi/5XywKLjCD3g/default.jpg');
```
Here, we mock (fake) the YouTubeService class and its getThumbnailByID method to which we pass the parameter. Then, we tell that it should be run only once in that request and should return the desired result.

It might seem complex at first glance, but what it means is it will NOT actually execute the method, replacing the call to that method with the result we hardcode. In our example, everything inside getThumbnailByID() will be ignored.

So in other words, we are not testing the Youtube API itself, we are testing that our application Service class method would be executed during the request lifecycle. The whole test could look like this:
```
class ProductsTest extends TestCase
{
    use RefreshDatabase;
 
    private function createUser($isAdmin = false)
    {
        return User::factory()->create([
            'email' => $isAdmin ? 'admin@admin.com' : 'admin@user.com',
            'is_admin' => $isAdmin
        ]);
    }
 
    public function test_store_product_exists_in_database()
    {
        $adminUser = $this->createUser(isAdmin: true);
 
        $this->mock(YouTubeService::class)
            ->shouldReceive('getThumbnailByID')
            ->with('5XywKLjCD3g')
            ->once()
            ->andReturn('https://i.ytimg.com/vi/5XywKLjCD3g/default.jpg');
 
        $product = [
            'name' => 'New product',
            'price' => 123,
            'youtube_id' => '5XywKLjCD3g',
        ];
 
        $response = $this->followingRedirects()->actingAs($adminUser)->post('products', $product);
 
        $response->assertRedirect('products');
        $this->assertDatabaseHas('products', $product);
 
        $latestProduct = Product::latest()->first();
        $this->assertEquals($latestProduct->name, $product['name']);
        $this->assertEquals($latestProduct->price, $product['price']);
    }
```
Notice we're not calling the getThumbnailByID() directly, we're making a POST request to the URL that points to the Controller that would call that Service method.

## Option 3. No Faking: Sandbox API Keys
Now, let's cover the case where you DO want to test the external service and you do want to make the request to that service, without faking it. For this, we will check the example from the open-source package Laravel Cashier.

In the main FeatureTestCase.php file, we have the stripe() method which creates the Cashier Stripe object with a real API key of the stripe.
```
abstract class FeatureTestCase extends TestCase
{
    use RefreshDatabase;
 
    //
    protected static function stripe(array $options = []): StripeClient
    {
        return Cashier::stripe(array_merge(['api_key' => getenv('STRIPE_SECRET')], $options));
    }
    //
}
```
Important point: that Stripe API Key should be from your testing environment on Stripe. In your external API, it may be called a Sandbox, or a Developer Account, or maybe you could even create a totally separate account just for testing purposes.

Then, you set those API keys in the automated test itself, or globally in the .env file of your testing environment on the server, or wherever you emulate that test - for example, in GitHub actions. Then, the package Cashier Stripe executes all the Stripe requests, but with your testing account.

Then, for the test itself, let's take a look at an example from CheckoutTest
```
class CheckoutTest extends FeatureTestCase
{
    public function test_customers_can_start_a_product_checkout_session()
    {
        $user = $this->createCustomer('customers_can_start_a_product_checkout_session');
 
        $shirtPrice = self::stripe()->prices->create([
            'currency' => 'USD',
            'product_data' => [
                'name' => 'T-shirt',
            ],
            'unit_amount' => 1500,
        ]);
 
        $carPrice = self::stripe()->prices->create([
            'currency' => 'USD',
            'product_data' => [
                'name' => 'Car',
            ],
            'unit_amount' => 30000,
        ]);
 
        $items = [$shirtPrice->id => 5, $carPrice->id];
 
        $checkout = $user->checkout($items, [
            'success_url' => 'http://example.com',
            'cancel_url' => 'http://example.com',
        ]);
 
        $this->assertInstanceOf(Checkout::class, $checkout);
    }
}
```
What is happening here?

- self::stripe() is called multiple times
- With that testing-environment object, we create the prices twice
- With the same object, we perform the checkout.
In other words, you need to set up such tests in a way that you set up your testing sandbox credentials once in some global config, constructor method, or similar. Then, your real code would just use the real API production keys, and the testing suite would replace them with sandbox/testing credentials.

## A Little Clearer Now?
With those three different real-life examples, I wanted to show you how to still perform automated testing, even if your request relies on some external API.

The choice is yours on which option to choose, and it may totally depend on the projects you work with.

For more lessons about automated testing, check out my two courses:

Laravel Testing For Beginners: PHPUnit, Pest, TDD (27 lessons, 2 h 06 min)
Advanced Laravel Testing (33 lessons, 1 h 37 min)
