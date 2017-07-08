# Rinvex Subscribable

**Rinvex Subscribable** is a flexible plans and subscription management system for Laravel, with the required tools to run your SAAS like services efficiently. It's simple architecture, accompanied by powerful underlying to afford solid platform for your business.

[![Packagist](https://img.shields.io/packagist/v/rinvex/subscribable.svg?label=Packagist&style=flat-square)](https://packagist.org/packages/rinvex/subscribable)
[![VersionEye Dependencies](https://img.shields.io/versioneye/d/php/rinvex:subscribable.svg?label=Dependencies&style=flat-square)](https://www.versioneye.com/php/rinvex:subscribable/)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/rinvex/subscribable.svg?label=Scrutinizer&style=flat-square)](https://scrutinizer-ci.com/g/rinvex/subscribable/)
[![Code Climate](https://img.shields.io/codeclimate/github/rinvex/subscribable.svg?label=CodeClimate&style=flat-square)](https://codeclimate.com/github/rinvex/subscribable)
[![Travis](https://img.shields.io/travis/rinvex/subscribable.svg?label=TravisCI&style=flat-square)](https://travis-ci.org/rinvex/subscribable)
[![SensioLabs Insight](https://img.shields.io/sensiolabs/i/81ab6092-1c20-4eb3-b180-2fd13114384e.svg?label=SensioLabs&style=flat-square)](https://insight.sensiolabs.com/projects/81ab6092-1c20-4eb3-b180-2fd13114384e)
[![StyleCI](https://styleci.io/repos/93313402/shield)](https://styleci.io/repos/93313402)
[![License](https://img.shields.io/packagist/l/rinvex/subscribable.svg?label=License&style=flat-square)](https://github.com/rinvex/subscribable/blob/develop/LICENSE)


## Considerations

- Payments are out of scope for this package.
- You may want to extend some of the core models, in case you need to override the logic behind some helper methods like `renew()`, `cancel()` etc. E.g.: when cancelling a subscription you may want to also cancel the recurring payment attached.


## Installation

1. Install the package via composer:
    ```shell
    composer require rinvex/subscribable
    ```

2. Execute migrations via the following command:
    ```shell
    php artisan migrate --path="vendor/rinvex/subscribable/database/migrations"
    ```

3. Add the following service provider to the `'providers'` array inside `app/config/app.php`:
    ```php
    Rinvex\Subscribable\Providers\SubscribableServiceProvider::class,
    ```

   And then you can publish the migrations by running the following command:
    ```shell
    php artisan vendor:publish --tag="migrations" --provider="Rinvex\Subscribable\Providers\SubscribableServiceProvider"
    ```

   And also you can publish the config by running the following command:
    ```shell
    php artisan vendor:publish --tag="config" --provider="Rinvex\Subscribable\Providers\SubscribableServiceProvider"
    ```

4. Done!


## Usage

### Add Subscribable to User model

**Rinvex Subscribable** has been specially made for Eloquent and simplicity has been taken very serious as in any other Laravel related aspect. To add Subscribable functionality to your User model just use the `\Rinvex\Subscribable\Traits\PlanSubscriber` trait like this:

```php
namespace App\Models;

use Rinvex\Subscribable\Traits\PlanSubscriber;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    use PlanSubscriber;
```

That's it, we only have to use that trait in our User model! Now your users may subscribe to plans.

### Create a Plan
```php
use Rinvex\Subscribable\Models\Plan;
use Rinvex\Subscribable\Models\PlanFeature;

$plan = Plan::create([
    'name' => 'Pro',
    'description' => 'Pro plan',
    'price' => 9.99,
    'invoice_period' => 1,
    'invoice_interval' => 'month',
    'trial_period' => 15,
    'trial_interval' => 'd',
    'sort_order' => 1,
    'currency' => 'USD',
]);

// Create multiple plan features at once
$plan->features()->saveMany([
    new PlanFeature(['name' => 'listings', 'value' => 50, 'sort_order' => 1]),
    new PlanFeature(['name' => 'pictures_per_listing', 'value' => 10, 'sort_order' => 5]),
    new PlanFeature(['name' => 'listing_duration_days', 'value' => 30, 'sort_order' => 10, 'resettable_period' => 1, 'resettable_interval' => 'month']),
    new PlanFeature(['name' => 'listing_title_bold', 'value' => 'Y', 'sort_order' => 15])
]);
```

### Get Plan Details

You can query the plan for further details, using the intuitive API as follows:

```php
use Rinvex\Subscribable\Models\Plan;

$plan = Plan::find(1);

// Get all plan features                
$plan->features;

// Get all plan subscriptions
$plan->subscriptions;

// Check if the plan is free
$plan->isFree();

// Check if the plan has trial period
$plan->hasTrial();

// Check if the plan has grace period
$plan->hasGrace();
```

Both `$plan->features` and `$plan->subscriptions` are collections, driven from relationships, and thus you can query these relations as any normal Eloquent relationship. E.g. `$plan->features()->where('slug', 'listing_title_bold')->first()`.

### Get Feature Value 

Say you want to show the value of the feature _pictures_per_listing_ from above. You can do so in many ways:

```php
// Use the plan instance to get feature's value
$amountOfPictures = $plan->getFeatureBySlug('pictures_per_listing')->value;

// Query the feature itself directly
$amountOfPictures = PlanFeature::where('slug', 'pictures_per_listing')->first()->value;

// Get feature value through the subscription instance
$amountOfPictures = PlanSubscription::find(1)->getFeatureValue('pictures_per_listing');
```

### Create a Subscription

You can subscribe a user to a plan by using the `newSubscription()` function available in the `PlanSubscriber` trait. First, retrieve an instance of your subscriber model, which typically will be your user model and an instance of the plan your user is subscribing to. Once you have retrieved the model instance, you may use the `newSubscription` method to create the model's subscription.

```php
use Rinvex\Subscribable\Models\Plan;

$user = User::find(1);
$plan = Plan::find(1);

$user->newSubscription('main', $plan);
```

The first argument passed to `newSubscription` method should be the name of the subscription. If your application offer a single subscription, you might call this `main` or `primary`. The second argument is the plan instance your user is subscribing to.

### Change the Plan

You can change subscription plan easily as follows:

```php
use Rinvex\Subscribable\Models\PlanSubscription;

$plan = Plan::find(2);
$subscription = PlanSubscription::find(1);

// Change subscription plan
$subscription->changePlan($plan);
```

If both plans (current and new plan) have the same billing frequency (e.g., `invoice_period` and `invoice_interval`) the subscription will retain the same billing dates. If the plans don't have the same billing frequency, the subscription will have the new plan billing frequency, starting on the day of the change and _the subscription usage data will be cleared_. Also if the new plan has a trial period and it's a new subscription, the trial period will be applied.

### Feature Options

Plan features are great for fine tuning subscriptions, you can topup certain feature for X times of usage, so users may then use it only for that amount. Features also have the ability to be resettable and then it's usage could be expired too. See the following examples:

```php
use Carbon\Carbon;
use Rinvex\Subscribable\Models\PlanFeature;

// Find plan feature
$feature = PlanFeature::where('slug', 'listing_duration_days')->first();

// Get feature reset date
$feature->getResetDate(new Carbon());
```

### Subscription Feature Usage

There's multiple ways to determine the usage and ability of a particular feature in the user subscription, the most common one is `canUseFeature`:

The `canUseFeature` method returns `true` or `false` depending on multiple factors:

- Feature _is enabled_.
- Feature value isn't `0`/`false`/`NULL`.
- Or feature has remaining uses available.

```php
$user->subscription('main')->canUseFeature('listings');
```

Other feature methods on the user subscription instnace are:

- `getFeatureUsage`: returns how many times the user has used a particular feature.
- `getFeatureRemainings`: returns available uses for a particular feature.
- `getFeatureValue`: returns the feature value.

> All methods share the same signature: e.g. `$user->subscription('main')->getFeatureUsage('listings');`.

### Record Feature Usage

In order to effectively use the ability methods you will need to keep track of every usage of each feature (or at least those that require it). You may use the `recordFeatureUsage` method available through the user `subscription()` method:

```php
$user->subscription('main')->recordFeatureUsage('listings');
```

The `recordFeatureUsage` method accept 3 parameters: the first one is the feature's slug, the second one is the quantity of uses to add (default is `1`), and the third one indicates if the addition should be incremental (default behavior), when disabled the usage will be override by the quantity provided. E.g.:

```php
// Increment by 2
$user->subscription('main')->recordFeatureUsage('listings', 2);

// Override with 9
$user->subscription('main')->recordFeatureUsage('listings', 9, false);
```

### Reduce Feature Usage

Reducing the feature usage is _almost_ the same as incrementing it. Here we only _substract_ a given quantity (default is `1`) to the actual usage:

```php
$user->subscription('main')->reduceFeatureUsage('listings', 2);
```

### Clear The Subscription Usage Data

```php
$user->subscription('main')->usage()->delete();
```

### Check Subscription Status

For a subscription to be considered active _one of the following must be `true`_:

- Subscription has an active trial.
- Subscription `ends_at` is in the future.

```php
$user->subscribedTo($planId);
```

Alternatively you can use the following methods available in the subscription model:

```php
$user->subscription('main')->active();
$user->subscription('main')->canceled();
$user->subscription('main')->ended();
$user->subscription('main')->onTrial();
```

> Canceled subscriptions with an active trial or `ends_at` in the future are considered active.

### Renew a Subscription

To renew a subscription you may use the `renew` method available in the subscription model. This will set a new `ends_at` date based on the selected plan and _will clear the usage data_ of the subscription.

```php
$user->subscription('main')->renew();
```

_Canceled subscriptions with an ended period can't be renewed._

### Cancel a Subscription

To cancel a subscription, simply use the `cancel` method on the user's subscription:

```php
$user->subscription('main')->cancel();
```

By default the subscription will remain active until the end of the period, you may pass `true` to end the subscription _immediately_:

```php
$user->subscription('main')->cancel(true);
```

### Scopes

#### Subscription Model

```php
use Rinvex\Subscribable\Models\PlanSubscription;

// Get subscriptions by plan:
$subscriptions = PlanSubscription::byPlanId($plan_id)->get();

// Get subscription by user:
$subscription = PlanSubscription::byUserId($user_id)->get();

// Get subscriptions with trial ending in 3 days:
$subscriptions = PlanSubscription::findEndingTrial(3)->get();

// Get subscriptions with ended trial:
$subscriptions = PlanSubscription::findEndedTrial()->get();

// Get subscriptions with period ending in 3 days:
$subscriptions = PlanSubscription::findEndingPeriod(3)->get();

// Get subscriptions with ended period:
$subscriptions = PlanSubscription::findEndedPeriod()->get();
```

### Models

**Rinvex Subscribable** uses 4 models:

```php
Rinvex\Subscribable\Models\Plan;
Rinvex\Subscribable\Models\PlanFeature;
Rinvex\Subscribable\Models\PlanSubscription;
Rinvex\Subscribable\Models\PlanSubscriptionUsage;
```


## Changelog

Refer to the [Changelog](CHANGELOG.md) for a full history of the project.


## Support

The following support channels are available at your fingertips:

- [Chat on Slack](http://chat.rinvex.com)
- [Help on Email](mailto:help@rinvex.com)
- [Follow on Twitter](https://twitter.com/rinvex)


## Contributing & Protocols

Thank you for considering contributing to this project! The contribution guide can be found in [CONTRIBUTING.md](CONTRIBUTING.md).

Bug reports, feature requests, and pull requests are very welcome.

- [Versioning](CONTRIBUTING.md#versioning)
- [Pull Requests](CONTRIBUTING.md#pull-requests)
- [Coding Standards](CONTRIBUTING.md#coding-standards)
- [Feature Requests](CONTRIBUTING.md#feature-requests)
- [Git Flow](CONTRIBUTING.md#git-flow)


## Security Vulnerabilities

If you discover a security vulnerability within this project, please send an e-mail to [help@rinvex.com](help@rinvex.com). All security vulnerabilities will be promptly addressed.


## About Rinvex

Rinvex is a software solutions startup, specialized in integrated enterprise solutions for SMEs established in Alexandria, Egypt since June 2016. We believe that our drive The Value, The Reach, and The Impact is what differentiates us and unleash the endless possibilities of our philosophy through the power of software. We like to call it Innovation At The Speed Of Life. That’s how we do our share of advancing humanity.


## License

This software is released under [The MIT License (MIT)](LICENSE).

(c) 2016-2017 Rinvex LLC, Some rights reserved.