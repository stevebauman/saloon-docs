# 🛸 Data Transfer Objects

When building API integrations, sometimes dealing with a raw response or a JSON response can be tedious and unpredictable. Data transfer objects are a good solution as they allow you to define a structure for a request and response. Saloon supports casting a response from an API request into a DTO.

### Defining DTOs from responses

In your request, extend the `createDtoFromResponse` method. Within this method, you get access to the `Response` object containing the response to cast into a data transfer object.

```php
<?php

use Saloon\Http\Request;
use Saloon\Http\Response;

class GetServerRequest extends Request
{
    // {...}
    
    public function createDtoFromResponse(Response $response): Server
    {
        $data = $response->json();
    
        return new Server(
            id: $data['id'],
            name: $data['name'],
            ipAddress: $data['ip'],
        );
    }
}
```

This is what the `Server` DTO looks like:

```php
<?php

class Server
{
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly string $ipAddress,
    ){}
}
```

### Retrieving your DTO from responses

Now we have defined our DTO on our request or connector, we can use the build in `dto` or `dtoOrFail` methods on our response class.

```php
<?php

$connector = new ForgeConnector;

$response = $connector->send(new GetServerRequest(id: 12345));

// Create a DTO even if the response was a failure
$server = $response->dto();

// Create a DTO or throw an exception if the response is not successful
$server = $response->dtoOrFail();
```

{% hint style="info" %}
Saloon will attempt to create a DTO from your response no matter the status of the response. This allows you to create "error" data transfer objects. If you don't want to use this functionality, you can use the `dtoOrFail` method which will throw a LogicException if the response is a failure. **You can customise what is considered a failed response** [**here**](handling-failures.md#customising-when-saloon-thinks-a-request-has-failed)**.**
{% endhint %}

### PHPStan / TypeHinting DTOs

Due to the way DTOs are created in Saloon, you can lose the type of DTO. This can reduce the developer experience. There are two ways to get around this issue.

You can use an inline-doc block above the `dto`/`dtoOrFail` method call.

```php
/** @var Server $server */
$server = $response->dto();
```

Or you can use the `createDtoFromResponse` method directly on your connector or request - which will use the return type specified.

```php
$server = $request->createDtoFromResponse($response);
```

### Using a DTO to populate request data

You may also allow your data transfer objects to go both ways, back into requests. You can do this easily with Saloon, just accept your DTO as an argument in the constructor of your request and then use the DTO to set default properties inside of the request.

{% tabs %}
{% tab title="Definition" %}
<pre class="language-php"><code class="lang-php">&#x3C;?php

use Saloon\Http\Request;
use Saloon\Enums\Method;
use Saloon\Http\Response;
use Saloon\Contracts\Body\HasBody;
use Saloon\Traits\Body\HasJsonBody;

class UpdateServerRequest extends Request implements HasBody
{
    use HasJsonBody;

    protected Method $method = Method::PUT;
    
    public function __construct(readonly protected Server $server)
    {
        //
    }
    
    public function resolveEndpoint(): string
    {
<strong>        return '/servers/' . $this->server->id;
</strong>    }
    
    protected function defaultBody(): array
    {
        return [
<strong>            'name' => $this->server->name,
</strong>        ];
    }
    
    public function createDtoFromResponse(Response $response): Server
    {
        $data = $response->json();
    
        return new Server(
            id: $data['id'],
            name: $data['name'],
            ipAddress: $data['ip'],
        );
    }
}
</code></pre>
{% endtab %}

{% tab title="Usage" %}
```php
<?php

$connector = new ForgeConnector;
$response = $connector->send(new GetServerRequest(id: 12345));
$server = $response->dto();

// Modify the server
$server->name = 'YEE-HAW-2';

// Update the server request

$response = $connector->send(new UpdateServerRequest($server));
```
{% endtab %}
{% endtabs %}

### Accessing the response from your DTO

Sometimes debugging a DTO can be difficult, especially if you have passed the data object through your application and no longer have access to the original `Response` that you created the DTO from. Saloon can inject the response into your data transfer object for you if you use the `HasResponse` trait and the `WithResponse` interface. Let's add it to our existing `Server` DTO.

<pre class="language-php"><code class="lang-php">&#x3C;?php

use Saloon\Http\Response;
use Saloon\Traits\Responses\HasResponse;
use Saloon\Contracts\DataObjects\WithResponse;

<strong>class Server implements WithResponse
</strong>{
<strong>    use HasResponse;
</strong>
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly string $ipAddress,
    ){}
}
</code></pre>

Now whenever we retrieve an instance of our data transfer object, you will be able to access the underlying response that was created with it!

```php
<?php

$server = $response->dto();

$response = $server->getResponse();
```
