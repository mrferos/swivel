# Zumba ***Swivel***

[![Build Status](https://travis-ci.org/zumba/swivel.svg?branch=master)](https://travis-ci.org/zumba/swivel)
[![Coverage Status](https://coveralls.io/repos/zumba/swivel/badge.svg)](https://coveralls.io/r/zumba/swivel)

***Swivel*** is a fresh spin on an old idea: [Feature Flags](http://en.wikipedia.org/wiki/Feature_toggle) (toggles, bits, switches, etc.).

**Important:** This software is still under initial development.  Not all features are complete, and the API may change.

Typical Feature Flags are *all* or *nothing*: either the feature is on for everyone, or it is off for everyone.

```php
// Old School Feature Flag

if ($flagIsOn) {
    // Do something new
} else {
    // Do something old
}
```

Typical Feature Flags are based on boolean conditionals with few abstractions (if this, then that).  Although powerful in their simplicity, this typically leads to increased [cyclomatic complexity](http://en.wikipedia.org/wiki/Cyclomatic_complexity) and eventual technical debt.

## Swivel is Different
***Swivel*** is fundamentally different from Typical Feature Flags in two ways:

* Features can be enabled for a subset of an application's users.
* Features are not simple conditionals; a developer defines one or more strategies (behaviors) and ***Swivel*** takes care of determining which strategy to use.

### Buckets
With ***Swivel***, users are separated into one of ten "buckets," allowing a feature to be enabled for a subset of users.  The advantages of this approach are clear:

* Deploying a new feature to 10% of users enables developers to catch unforeseen bugs/problems with new code without negatively affecting all users.  These kind of deployments are called [Canary Releases](http://martinfowler.com/bliki/CanaryRelease.html).  As soon as it is determined that new code is safe, roll out the new feature to more users in increments (30%, 50%, etc.); eventually the feature can enabled for all users, safely.
* A/B testing becomes a breeze.  Imagine running up to 9 versions of a new feature with one group kept in reserve as a control.  Is feature "A" negatively affecting revenue metrics for 10% of your users? No problem: turn it off and go with version "B" instead.  This is easy to do with ***Swivel***.

### Behaviors
Agile code needs to be simple and easy to change.  Typical Feature Flags allow developers to quickly iterate when business rules change or new features are implemented, but this can often lead to complex, under engineered, brittle blocks of code.

***Swivel*** encourages the developer to implement changes to business logic as independent, high level *strategies* rather than simple, low level deviations.

## Example: Quick Look

```php
$formula = $swivel->forFeature('AwesomeSauce')
    ->addBehavior('formulaSpicy', [$this, 'getNewSpicyFormula'])
    ->addBehavior('formulaSaucy', [$this, 'getNewSaucyFormula'])
    ->defaultBehavior([$this, 'getFormula'])
    ->execute();
```

## Getting Started

The first thing you'll want to do is generate a random number between 1 and 10 for each user in your application. We call this the user's "bucket" index.  This is what is used by ***Swivel*** to determine which features are enabled for that user.

**Note:** As a best practice, once a user is assigned to a bucket they should remain in that bucket forever. You'll want to store this value in a session or cookie like you would other basic user info.

Next, you'll need to create a map of features.  This map indicates which buckets should have certain features enabled.  Here is an example of a simple feature map:

```php
$map = [
    // This is a parent feature slug, arbitrarily named "Payment."
    // The "Payment" feature is enabled for users in buckets 4, 5, and 6
    'Payment' => [4,5,6],

    // This is a behavior slug.  It is a subset of the parent slug,
    // and it is only enabled for users in buckets 4 and 5
    'Payment.Test' => [4, 5],

    // Behavior slugs can be nested.
    // This one is only enabled for users in bucket 5.
    'Payment.Test.VersionA' => [5]
];
```

When your application starts, configure ***Swivel*** and create a new manager instance:

```php
// Get this value from the session or from persistent storage.
$userBucket = 5; // $_SESSION['bucket'];

// Get the feature map data from persistent storage.
$mapData = [ 'Feature' => [4,5,6], 'Feature.Test' => [4,5] ];

// Make a new configuration object
$config = new \Zumba\Swivel\Config($mapData, $userBucket);

// Make a new Swivel Manager.  This is your primary API to the Swivel library.
$swivel = new \Zumba\Swivel\Manager($config);
```

Way to go!  ***Swivel*** is now ready to use.

### Using Strategies
Now that you have a new ***Swivel*** manager created, you can start using it in your application.  To use ***Swivel*** you need to define behaviors for features of your code; ***Swivel*** will decide which behavior to execute based on the current user's bucket and the feature map you loaded in the configuration step.

#### Strategy Example

Say you have coded a new search algorithm for your website.  The Search feature of your site is integral to your business, so you only want to roll out the new algorithm to 10% of your users at first. You decide to only enable the algorithm for users in bucket `5`.  You configure ***Swivel*** and register it with your application:

```php
$map = [ 'Search' => [5], 'Search.NewAlgorithm' => [5] ];
$config = new \Zumba\Swivel\Config($map, $_SESSION['bucketIndex']);
$swivel = new \Zumba\Swivel\Manager($config);

// ServiceLocator is fictional in this example.  Use your own framework or repository to store the
// swivel instance.
ServiceLocator::add('Swivel', $swivel);
```

In your code to search the site, you define two distinct strategies, and tell ***Swivel*** about them:

```php
public function search($params = []) {
    $swivel = ServiceLocator::get('Swivel');
    return $swivel->forFeature('Search')
        ->addBehavior('NewAlgorithm', [$this, 'awesomeSearch'], [$params])
        ->defaultBehavior([$this, 'normalSearch'], [$params])
        ->execute();
}

public function normalSearch($params) {
    // Tried and True method.
}

public function awesomeSearch($params) {
    // Super cool new search method.
}
```

Now, when you call `search`, ***Swivel*** will execute `awesomeSearch` for users in bucket `5`, and `normalSearch` for all other users.

## ***Swivel*** API

### Zumba\Swivel\Config

#### constructor($map, $index, $logger)

Used to configure your ***Swivel*** Manager instance.

Param                         | Type       | Details
:-----------------------------|:-----------|:--------
**$map**<br/>*(optional)*    | *mixed*   | Can be one of the following:<br/><ul><li>`array` &mdash; an array of feature/behavior slugs as keys and enabled buckets as values.</li><li>`\Zumba\Swivel\MapInterface` &mdash; An instance of a configured feature map.</li><li>`\Zumba\Swivel\DriverInterface` &mdash; An instance of a ***Swivel*** driver that will build a `MapInterface` object.</li></ul>
**$index**<br/>*(optional)*  | *integer* | The user's predefined bucket index.  A number between 1 and 10.
**$logger**<br/>*(optional)* | `LoggerInterface` | An optional logger that implements `\Psr\Log\LoggerInterface`

### Zumba\Swivel\Manager

#### constructor($config)

This is the primary ***Swivel*** object that you will use in your app.

Param       | Type     | Details
:-----------|:---------|:--------
**$config** | `Config` | A `\Zumba\Swivel\Config` instance.

#### forFeature($slug)

Create a point of deviation in your code.  Returns a new `Zumba\Swivel\Builder` that accepts multiple behaviors, default behaviors, and executes the appropriate code for the user's bucket.

Param     | Type     | Details
:---------|:---------|:--------
**$slug** | *string* | The first section of a feature map slug.  i.e., for the feature slug `"Test.Version.Two"`, the `$slug` would be `"Test"`

#### invoke($slug, $a, $b)

Shorthand syntactic sugar for invoking a simple feature behavior.  Useful for ternary style code:

```php
// traditional
$result $newSearch ? $this->search() : $this->noOp();

// Zumba\Swivel\Manager::invoke
$result = $swivel->invoke('Search.New', [$this, 'search'], [$this, 'noOp']);
```

Param                   | Type       | Details
:-----------------------|:-----------|:--------
**$slug**               | *string*   | The first section of a feature map slug.  i.e., for the feature slug `"Test.Version.Two"`, the `$slug` would be `"Test"`
**$a**                  | *callable* | The strategy to execute if the `$slug` is enabled for the user's bucket.
**$b**<br/>*(optional)* | *callable* | The strategy to execute if the `$slug` is not enabled for the user's bucket.  If omitted, `invoke` will return `null` if the feature slug is not enabled.

### Zumba\Swivel\Builder

The `Builder` API is the primary way that you will write ***Swivel*** code.  You get a new instance of the `Builder` when you call `Manager::forFeature`.

#### addBehavior($slug, $strategy, $args)

Lazily adds a behavior to this feature that will only be executed if the feature is enabled for the user's bucket.

Param                      | Type       | Details
:--------------------------|:-----------|:--------
**$slug**                  | *string*   | The second section of a feature map slug.  i.e., for the feature slug `"Test.Version.Two"`, the `$slug` here would be `"Version.Two"`
**$strategy**              | *callable* | The strategy to execute if the `$slug` is enabled for the user's bucket.
**$args**<br/>*(optional)* | *array*    | Parameters to pass to the `$strategy` callable if it is executed.

#### defaultBehavior($strategy, $args)

Lazily adds a behavior to this feature that will only be executed if no other feature behaviors are enabled for the user's bucket.

Param                      | Type       | Details
:--------------------------|:-----------|:--------
**$strategy**              | *callable* | The strategy to execute if no other feature behaviors are enabled for the user's bucket.
**$args**<br/>*(optional)* | *array*    | Parameters to pass to the `$strategy` callable if it is executed.

#### execute()

Executes the appropriate behavior strategy based on the user's bucket.

#### noDefault()

If you do not need to define a default behavior to be executed when a feature is not enabled for a user's bucket, call `noDefault` on the `Builder`.  ***Swivel*** will throw a `\LogicException` if you neglect to define a default behavior and do not call `noDefault`.  Likewise, ***Swivel*** will throw a `\LogicException` if you call both `noDefault` and `defaultBehavior` on the same `Builder` instance.
