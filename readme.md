# Feed Generator

This library aims to simplify compliant feed generation for Shopping-Feed services.

### Requirements

- PHP version 5.5 or above
- PHP XML extension

### Installation

`composer require shoppingfeed/php-feed-generator`

### Overview

The component act as pipeline to convert any data set to compliant XML output, to a file or any other destination.

From the user point of view, it consist on mapping your data to a `ShoppingFeed\FeedProduct` object.
The library take cares of the rest : formatting data and write valid XML.


### Getting Started

A minimal valid XML Product item requires the following properties:

- A `reference` (aka SKU)
- A `name`
- A `price`
- A `quantity`

Your data source must contains those information, or you can hardcode some of them during feed generator.

First, you should creates an instance of `ShoppingFeed\Feed\ProductFeed` and configure it according your needs

```php
<?php
namespace ShoppingFeed\Feed;

$feed = new ProductFeed();
```

By default, XML is wrote to the standard output, but you can specify an uri, and pointing it to a file 

```php
<?php
namespace ShoppingFeed\Feed;

$feed = new ProductFeed();
$feed->setUri('file://my-feed.xml');
```

The `ProductFeed` accept some settings or useful information, like the platform or application for which the feed has been generated.
We recommand to always specify this setting, because it can help us to debug and provide appropriate support

```php
<?php
namespace ShoppingFeed\Feed;

$feed = (new ProductFeed)
    ->setUri('file://my-feed.xml')
    ->setPlatform('Magento', '2.2.1');

# Set any extra vendor attributes.
$feed    
    ->setAttribute('storeName', 'my great store')
    ->setAttribute('storeUrl', 'http://my-greate-store.com');
```

### Basic Example

Once the feed instance is properly configured, you must provide at least one `mapper` and a dataset to run the feed generation against

```php
<?php
namespace ShoppingFeed\Feed;

$items[0] = ['sku' => 1, 'title' => 'Product 1', 'price' => 5.99, 'quantity' => 3];
$items[1] = ['sku' => 2, 'title' => 'Product 2', 'price' => 12.99, 'quantity' => 6];

$feed = (new ProductFeed)->setPlatform('Magento', '2.2.1');

# Mappers are responsible to convert your data format to hydrated product
$feed->addMapper(function(array $item, Product\Product $product) {
    $product
        ->setName($item['title'])
        ->setReference($item['sku'])
        ->setPrice($item['price'])
        ->setQuantity($item['quantity']);
});

# now generates the feed with $items collection
$feed->write($items);
```
That's all ! Put this code in a script then run it, XML should appear to your output (browser or terminal).


## Processing overview

This schema describe to loop pipeline execution order

```
-> exec processors[] -> exec filters[] -> exec mappers[] -> write ->
|                                                                  |
<-------------------------------------------------------------------
```

### Processors

In some case, you may need to pre-process data before to map them.

This can be achieved in mappers or in your dataset, but sometimes things have to be separated, so you can register processors that are executed before mappers, and prepare your data before the mapping process.

In this example, we try to harcode quantity to zero when not specified in item, to populate required `quantity` field later

```php
<?php
namespace ShoppingFeed\Feed;

$feed = new ProductFeed;

$feed->addProcessor(function(array $item) {
    if (! isset($item['quantity'])) {
        $item['quantity'] = 0;
    }
    # modified data must be returned
    return $item;
});
 
$feed->addMapper(function(array $item, Product\Product $product) {
    # Operation is now "safe", we have a valid quantity integer here
    $product->setQuantity($item['quantity']);
});
```

As mappers, you can register any processors as you want, but processors :

- Only accept $item as argument
- Expects return value
- Returned value is used for the next stage (next processor or next mapper)


### Filters

Filters are designed discard some items from the feed.

Filters are executed **after** processors, because item must be completely filled before to make the decision to keep it or not.

Expected return value is a boolean, where:

- `TRUE`  : the item is passed to mappers
- `FALSE` : the item is ignored

### Mappers

As stated above, at least one mapper must be registered, this is where you populate the `Product` instance, which is later converted to XML by the library

The `addMapper` method accept any [callable type](http://php.net/manual/en/language.types.callable.php), like functions or invokable objects.

Mappers are inkoked on each iteration over the collection you provided in the `generate` method, with the following arguments

- `(mixed $item, ShoppingFeed\FeedProduct\Product $product)`

where:

- `$item` is your data
- `$product` is the object to populate with `$item` data

Note that there is *no expected return value* from your callback

#### How mapper are invoked ?

You can provide any mappers as you want, they are executed in FIFO (First registered, First executed) mode.
The ability to register more than once mapper can helps to keep your code organized as you want, and there is no particular performances hints when registering many mapper.

As an example of organisation, you can register 1 mapper for product, and 1 for its variations

```php
<?php
namespace ShoppingFeed\Feed;

$feed = new ProductFeed;

# Populate properties
$feed->addMapper(function(array $item, Product\Product $product) {
    $product
        ->setName($item['title'])
        ->setReference($item['sku'])
        ->setPrice($item['price'])
        ->setQuantity($item['quantity']);
});

# Populate product's variations. Product properties are already populated by the previous mapper
$feed->addMapper(function(array $item, Product\Product $product) {
    foreach ($item['declinations'] as $item) {
        $variation = $product->createVariation();
        $variation
            ->setReference($item['sku'])
            ->setPrice($item['price'])
            ->setQuantity($item['quantity']);
    }
});
```


```php
<?php
namespace ShoppingFeed\Feed;

$feed = new ProductFeed;

# Ignore all items with undefined quantity
$feed->addFilter(function(array $item) {
   return isset($item['quantity']);
});

# Ignore all items prices above 10
$feed->addFilter(function(array $item) {
   return $item['price'] <= 10;
});

# only items that match previous filters conditions are considered by mappers
$feed->addMapper(function(array $item, Product\Product $product) {
    // do some stuff
});
```

### Performances Considerations

Generating large XML feed can be a very long process, so our advices in this area are:

- Run feed generation offline from a command line / cron : PHP [set max_exection_time to 0](http://php.net/manual/en/info.configuration.php#ini.max-execution-time) in this mode
- Try to generate feed on different machine than web-store, or when the traffic is limited : it may impact the your visitor experience by using too much resources  
- Limit the number of SQL requests : avoid running request in the loop
- Paginate your results on large dataset, this will limit memory consumption and network traffic per request

Internally, the library uses `XmlReader` / `XmlWriter` to limit memory consumption. Product objects and generated XML are flushed from memory after each iteration.
This guaranty that memory usage will not increase with the number of products to write, but only depends on the "size" of each products.


### Execute test command

The script will generates random products

```bash
php tests/functional/simple.php <file> <number-of-products>

# example:
php tests/functional/simple.php feed.xml 1000
```   