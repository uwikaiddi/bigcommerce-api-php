# BigCommerce API Client

PHP client for connecting to the BigCommerce V2 REST API.

To find out more, visit the official documentation website:
http://developer.bigcommerce.com/

[![Build Status](https://travis-ci.org/bigcommerce/bigcommerce-api-php.png?branch=master)](https://travis-ci.org/bigcommerce/bigcommerce-api-php)
[![Coverage Status](https://coveralls.io/repos/bigcommerce/bigcommerce-api-php/badge.png?branch=master)](https://coveralls.io/r/bigcommerce/bigcommerce-api-php?branch=master)
[![Dependency Status](https://www.versioneye.com/package/php--bigcommerce--api/badge.png)](https://www.versioneye.com/package/php--bigcommerce--api)

[![Latest Stable Version](https://poser.pugx.org/bigcommerce/api/v/stable.png)](https://packagist.org/packages/bigcommerce/api)
[![Total Downloads](https://poser.pugx.org/bigcommerce/api/downloads.png)](https://packagist.org/packages/bigcommerce/api)

## Contents

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Requirements](#requirements)
- [Installation](#installation)
	- [Composer](#composer)
- [Configuration](#configuration)
- [Usage](#usage)
	- [Accessing collections and resources (GET)](#accessing-collections-and-resources-get)
	- [Updating existing resources (PUT)](#updating-existing-resources-put)
	- [Creating new resources (POST)](#creating-new-resources-post)
	- [Deleting resources (DELETE)](#deleting-resources-delete)
	- [Paging and Filtering](#paging-and-filtering)
- [Additional examples](#additional-examples)

<!-- /TOC -->

## Requirements

- PHP 7.0 or greater
- cUrl extension enabled

**To connect to the API with basic authorization, you need the following:**

- **store-hash:** Secure URL pointing to a Bigcommerce store
- **client_id:** Username of an authorized admin user of the store
- **access_token:** API key for the admin user
- **client_secret:** Generated password from the authorization request

To generate an API key/access token:
1. Go to **Control Panel** > **Advanced Settings** > **API Accounts**.
2. On the Store API Accounts page, click **Create API Account** and select "Create V2/V3 API Token."

## Installation

The BigCommerce PHP API client can be installed using [Composer](https://packagist.org/packages/zendesk/zendesk_api_client_php).

### Composer

Use the following Composer command to install the API client from [the Bigcommerce vendor on Packagist](https://packagist.org/packages/bigcommerce/api):

~~~shell
 $ composer require bigcommerce/api
 $ composer update
~~~

You can also install Composer for your specific project by running the following in the library folder.

~~~shell
 $ curl -sS https://getcomposer.org/installer | php
 $ php composer.phar install
 $ composer install
~~~

## Configuration

To instantiate the PHP client, create a factory object:

~~~php
$api_url       = 'https://api.bigcommerce.com/stores/your-store-hash';
$client_id     = 'your-client-id';
$access_token  = 'your-access-token';
$client_secret = 'your-client-secret';

$api = new \BigCommerce\Api\ApiFactory( $api_url, $client_id, $access_token, $client_secret );
~~~

You can find the required values in the BigCommerce admin when you create a new access token, or create an application that uses [OAuth to request an access token](#).

You can instantiate the following APIs using the factory object:

~~~php
$api->abandoned_cart();
$api->cart();
$api->catalog();
$api->channels();
$api->orders();
$api->price_lists();
$api->pricing();
$api->scripts();
$api->sites();
$api->store();
$api->subscribers();
$api->themes();
$api->transactions();
$api->widgets();
$api->wishlists();
~~~

## Usage

The following section contains examples of the calls available using this API client. For more information on available functions, see the [source file](https://github.com/moderntribe/bigcommerce-api-php/tree/master/src).

### Accessing collections and resources (GET)

~~~php
$api     = new \BigCommerce\Api\ApiFactory( $api_url, $client_id, $access_token, $client_secret );
$catalog = $api->catalog();

$product_ids = [ 100, 101, 102 ];

try {
	/*
	 * List of request parameters and response properties available at
	 * https://developer.bigcommerce.com/api-reference/catalog/catalog-api/products/getproducts
	 */
	$products_response = $catalog->getProducts( [
		'id:in'   => $product_ids,
		'include' => [ 'variants', 'custom_fields', 'images', 'bulk_pricing_rules', 'options', 'modifiers' ],
	] );
} catch ( \BigCommerce\Api\ApiException $e ) {
	$error_message = $e->getMessage();
	$error_body    = $e->getResponseBody();
	$error_headers = $e->getResponseHeaders();
	// do something with the error
	return;
}

$found_product_ids = array_map( function( \BigCommerce\Api\Model\Product $product ) {
	return $product->getId();
}, $products_response->getData() );
~~~

### Updating existing resources (PUT)

~~~php
use BigCommerce\Api\Model\Variant;
use BigCommerce\Api\Model\VariantPut;
use BigCommerce\Api\Model\VariantResponse;

$api     = new \BigCommerce\Api\ApiFactory( $api_url, $client_id, $access_token, $client_secret );
$catalog = $api->catalog();

$product_id = 101;

try {
	$variants_responses = [];
	$page = 0;
	$per_page = 100;

	// Request all variants. There may be more than one page of results.
	do {
		$page ++;
		/*
		 * List of request parameters and response properties available at
		 * https://developer.bigcommerce.com/api-reference/catalog/catalog-api/product-variants/getvariantsbyproductid
		 */
		$variants_response = $catalog->getVariantsByProductId( $product_id, [
			'page'  => $page,
			'limit' => $per_page,
		] );
		$max_pages = $variants_response->getMeta()->getPagination()->getTotalPages();
		$variants_responses[] = $variants_response;
	} while ( $page < $max_pages );

	// Merge all the responses into one array
	$variants = array_merge( ...array_map( function( VariantResponse $response ) {
		return $response->getData();
	}, $variants_responses ) );

	// Increase inventory of each variant by 5
	array_map( function( Variant $variant ) use ( $catalog ) {
		$inventory_level = $variant->getInventoryLevel() + 5;

		$request = new VariantPut( $variant->get() );
		$request->setInventoryLevel( $inventory_level );

		$catalog->updateVariant( $variant->getProductId(), $variant->getId(), $request );
	}, $variants );

} catch ( \BigCommerce\Api\ApiException $e ) {
	$error_message = $e->getMessage();
	$error_body    = $e->getResponseBody();
	$error_headers = $e->getResponseHeaders();
	// do something with the error
	return;
}
~~~

### Creating new resources (POST)

~~~php
use BigCommerce\Api\Model\ModifierPost;
use BigCommerce\Api\Model\ProductPost;

$api     = new \BigCommerce\Api\ApiFactory( $api_url, $client_id, $access_token, $client_secret );
$catalog = $api->catalog();

try {

	/*
	 * List of request parameters and response properties available at
	 * https://developer.bigcommerce.com/api-reference/catalog/catalog-api/products/createproduct
	 */
	$product_request = new ProductPost( [
		'name'   => 'BigCommerce Coffee Mug',
		'price'  => '10.00',
		'weight' => 4,
		'type'   => 'physical',
		'variants' => [
			[
				'sku'           => 'MUG-BLUE',
				'option_values' => [
					'option_display_name' => 'Mug Color',
					'label'               => 'Blue',
				],
			],
			[
				'sku'           => 'MUG-GREY',
				'option_values' => [
					'option_display_name' => 'Mug Color',
					'label'               => 'Grey',
				],
			],
		]
	] );

	$product_response = $catalog->createProduct( $product_request );

	$product_id = $product_response->getData()->getId();

	/*
	 * List of request parameters and response properties available at
	 * https://developer.bigcommerce.com/api-reference/catalog/catalog-api/product-modifiers/createmodifier
	 */
	$modifier_request = new ModifierPost( [
		'type'         => 'text',
		'required'     => false,
		'display_name' => 'Custom Message',
	] );

	$catalog->createModifier( $product_id, $modifier_request );

} catch ( \BigCommerce\Api\ApiException $e ) {
	$error_message = $e->getMessage();
	$error_body    = $e->getResponseBody();
	$error_headers = $e->getResponseHeaders();
	// do something with the error
	return;
}
~~~

### Deleting resources (DELETE)

~~~php
$api     = new \BigCommerce\Api\ApiFactory( $api_url, $client_id, $access_token, $client_secret );
$catalog = $api->catalog();

$product_ids = [ 100, 101, 102 ];

try {
	/*
	 * List of request parameters and response properties available at
	 * https://developer.bigcommerce.com/api-reference/catalog/catalog-api/products/getproducts
	 */
	$products_response = $catalog->deleteVariantById( $product_id, $variant_id );
} catch ( \BigCommerce\Api\ApiException $e ) {
	$error_message = $e->getMessage();
	$error_body    = $e->getResponseBody();
	$error_headers = $e->getResponseHeaders();
	// do something with the error
	return;
}

$found_product_ids = array_map( function( \BigCommerce\Api\Model\Product $product ) {
	return $product->getId();
}, $products_response->getData() );
~~~

### Paging and Filtering

All the default collection methods support paging, by passing
the page number to the method as an integer:

~~~php
$products = Bigcommerce::getProducts(3);
~~~
If you require more specific numbering and paging, you can explicitly specify
a limit parameter:

~~~php
$filter = array("page" => 3, "limit" => 30);

$products = Bigcommerce::getProducts($filter);
~~~

To filter a collection, you can also pass parameters to filter by as key-value
pairs:

~~~php
$filter = array("is_featured" => true);

$featured = Bigcommerce::getProducts($filter);
~~~
See the API documentation for each resource for a list of supported filter
parameters.

## Additional examples
- [Add a route to a site](#)
- [Add a product to a cart](#)
- [Create a widget](#)
