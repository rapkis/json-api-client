# { json:api } Client

[![PHP from Packagist](https://img.shields.io/packagist/php-v/swisnl/jsonapi.svg)](https://packagist.org/packages/swisnl/jsonapi)
[![Latest Version on Packagist](https://img.shields.io/packagist/v/swisnl/jsonapi.svg)](https://packagist.org/packages/swisnl/jsonapi)
[![Software License](https://img.shields.io/packagist/l/swisnl/jsonapi.svg)](LICENSE)
[![Run Status](https://api.shippable.com/projects/59ae7fe68a02980600016631/badge?branch=master)](https://app.shippable.com/github/swisnl/jsonapi)
[![Coverage Badge](https://api.shippable.com/projects/59ae7fe68a02980600016631/coverageBadge?branch=master)](https://app.shippable.com/github/swisnl/jsonapi) 

A PHP package for mapping remote [{json:api}](http://jsonapi.org/) resources to Eloquent like models and collections.


## Installation

``` bash
composer require swisnl/jsonapi
```

### HTTP Client

We are decoupled from any HTTP messaging client with the help of [PHP-HTTP](http://php-http.org/).
This requires another package providing [php-http/client-implementation](https://packagist.org/providers/php-http/client-implementation).
To use Guzzle 6, for example, simply require `php-http/guzzle6-adapter`:

``` bash
composer require php-http/guzzle6-adapter
```

### Laravel Service Provider

If you are using Laravel < 5.5 or have disabled package auto discover, you must add the service provider to your `config/app.php` file:

``` php
'providers' => [
    ...,
    \Swis\JsonApi\Client\Providers\ServiceProvider::class,
],
```


## Getting started

You can simply require the client as a dependency and use it in your class.
You can, for example, make a repository that uses the DocumentClient:

``` php
use Swis\JsonApi\Interfaces\DocumentClientInterface;
use Swis\JsonApi\Interfaces\ItemDocumentInterface;

class BlogRepository
{
    protected $client;

    public function __construct(DocumentClientInterface $client)
    {
        $this->client = $client;
    }

    public function all(array $parameters = [])
    {
        return $this->client->get('blogs?'.http_build_query($parameters));
    }

    public function create(ItemDocumentInterface $document, array $parameters = [])
    {
        return $this->client->post('blogs?'.http_build_query($parameters), $document);
    }

    public function find(string $id, array $parameters = [])
    {
        return $this->client->get('blogs/'.urlencode($id).'?'.http_build_query($parameters));
    }

    public function update(ItemDocumentInterface $document, array $parameters = [])
    {
        return $this->client->patch('blogs/'.urlencode($document->getData()->getId()).'?'.http_build_query($parameters), $document);
    }

    public function delete(string $id, array $parameters = [])
    {
        return $this->client->delete('blogs/'.urlencode($id).'?'.http_build_query($parameters));
    }
}
```

## Configuration

The following is the default configuration provided by this package:

``` php
return [
    /*
    |--------------------------------------------------------------------------
    | Base URI
    |--------------------------------------------------------------------------
    |
    | Specify a base uri which will be prepended to every URI.
    |
    | Default: empty string
    |
    */
    'base_uri' => '',
];
```
        
### Publish Configuration

If you would like to make changes to the default configuration, publish and edit the configuration file:

``` bash
php artisan vendor:publish --provider="Swis\JsonApi\Client\Providers\ServiceProvider" --tag="config"
```


## Package flow
   
Short description
   
### DocumentClient

### Client


## Item models

By default, all items are an instance of `JenssegersItem`.
You can define your own models by extending the `JenssegersItem` or by implementing the `\Swis\JsonApi\Client\Interfaces\ItemInterface` yourself.
This can be useful if you want to define, for example, hidden attributes, casts or get/set mutators.
If you use custom models, you must register them with the [TypeMapper](#typemapper).

### JenssegersItem

The `JenssegersItem` extends [jenssegers/model](https://github.com/jenssegers/model), which provides a Laravel Eloquent-like base class.
Please see it's documentation about the features it provides.
On top of that, this package has implemented Laravel Eloquent-like relations.

#### Relations

The relations are basic variants of [Laravel Eloquent relationships](https://laravel.com/docs/eloquent-relationships) and provide a fluent interface to retrieve the related items.
There are currently four relations available:

 * `HasOneRelation`
 * `HasManyRelation`
 * `MorphToRelation`
 * `MorphToManyRelation`

Please see the following example about defining the relationships:

``` php
use Swis\JsonApi\Items\JenssegersItem;

class AuthorItem extends JenssegersItem
{
    protected $type = 'author';

    public function blogs()
    {
        return $this->hasMany(BlogItem::class);
    }
}

class BlogItem extends JenssegersItem
{
    protected $type = 'blog';

    public function author()
    {
        return $this->hasOne(AuthorItem::class);
    }
}
```

### EloquentItem (EXPERIMENTAL)

The `EloquentItem` directly extends the [Laravel Eloquent Model](https://laravel.com/docs/eloquent) and uses the provided relations.
Please note that this implementation is experimental and should not be used in production!


## Collections

This package uses [Laravel Collections](https://laravel.com/docs/collections) as a wrapper for item arrays.


## TypeMapper

All custom models must be registered with the `TypeMapper`.
This `TypeMapper` maps, as the name suggests, JSON API types to custom [item models](#item-models).

### Mapping types

You can manually register items with the `\Swis\JsonApi\Client\TypeMapper` or use the supplied `\Swis\JsonApi\Client\Providers\TypeMapperServiceProvider`:

``` php
use Swis\JsonApi\Providers\TypeMapperServiceProvider as BaseTypeMapperServiceProvider;

class TypeMapperServiceProvider extends BaseTypeMapperServiceProvider
{
    /**
     * A list of class names implementing \Swis\JsonApi\Client\Interfaces\ItemInterface.
     *
     * @var string[]
     */
    protected $items = [
        \App\Items\Author::class,
        \App\Items\Blog::class,
        \App\Items\Comment::class,
    ];
}
```


## Service Provider

The `\Swis\JsonApi\Client\Providers\ServiceProvider` registers the `TypeMapper`, `JsonApi\Parser` and both clients; `Client` and `DocumentClient`.
Each section can be overwritten to allow extended customization.

### Bind TypeMapper

The service provider registers the `TypeMapper` as a singleton so your entire application has the same mappings available.

### Bind Clients

The service provider registers the `Client` and `DocumentClient` to your application.
By default it uses [php-http/discovery](https://github.com/php-http/discovery) to find a HTTP client and message factory.
You can specify your own HTTP client by overwriting the `getHttpClient()` method.
This method should return a client that implements HTTPlug.
It is the perfect place to add extra options to your HTTP client or register a mock HTTP client for your tests:

``` php
protected function getHttpClient(): HttpClient
{
    if (app()->environment('testing')) {
        return new \Swis\Http\Fixture\Client(
            new \Swis\Http\Fixture\ResponseBuilder('/path/to/fixtures');
        );
    }

    return \Http\Adapter\Guzzle6\Client::createWithConfig(
        [
            'timeout' => 2,
        ]
    );
}
```

If you register your own service provider and use package auto discover, don't forget to exclude this package in your `package.json`:

``` json
"extra": {
  "laravel": {
    "dont-discover": [
      "swisnl/jsonapi"
    ]
  }
},
```


## Todo

 - [ ] CHANGELOG.MD
 - [ ] Increase test coverage


## Testing

``` bash
composer test
```


## Security

If you discover any security related issues, please email service@swis.nl instead of using the issue tracker.


## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.