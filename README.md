# Laravel Paystack

[![Latest Stable Version](https://poser.pugx.org/sajjadhossainshohag/laravel-paystack/v/stable.svg)](https://packagist.org/packages/sajjadhossainshohag/laravel-paystack)
[![License](https://poser.pugx.org/sajjadhossainshohag/laravel-paystack/license.svg)](LICENSE.md)

[![Quality Score](https://img.shields.io/scrutinizer/g/sajjadhossainshohag/laravel-paystack.svg?style=flat-square)](https://scrutinizer-ci.com/g/sajjadhossainshohag/laravel-paystack)
[![Total Downloads](https://img.shields.io/packagist/dt/sajjadhossainshohag/laravel-paystack.svg?style=flat-square)](https://packagist.org/packages/sajjadhossainshohag/laravel-paystack)

> This package is a modernized fork of unicodeveloper/laravel-paystack.

## Installation

[PHP](https://php.net) 5.4+ or [HHVM](http://hhvm.com) 3.3+, and [Composer](https://getcomposer.org) are required.

To get the latest version of Laravel Paystack, simply require it

```bash
composer require sajjadhossainshohag/laravel-paystack
```

Or add the following line to the require block of your `composer.json` file.

```
"sajjadhossainshohag/laravel-paystack": "1.0.*"
```

You'll then need to run `composer install` or `composer update` to download it and have the autoloader updated.



Once Laravel Paystack is installed, you need to register the service provider. Open up `config/app.php` and add the following to the `providers` key.

```php
'providers' => [
    ...
    Sajjadhossainshohag\Paystack\PaystackServiceProvider::class,
    ...
]
```

> If you use **Laravel >= 5.5** you can skip this step and go to [**`configuration`**](https://github.com/sajjadhossainshohag/laravel-paystack#configuration)

* `Sajjadhossainshohag\Paystack\PaystackServiceProvider::class`

Also, register the Facade like so:

```php
'aliases' => [
    ...
    'Paystack' => Sajjadhossainshohag\Paystack\Facades\Paystack::class,
    ...
]
```

## Configuration

You can publish the configuration file using this command:

```bash
php artisan vendor:publish --provider="Sajjadhossainshohag\Paystack\PaystackServiceProvider"
```

A configuration file named `paystack.php` with some sensible defaults will be placed in your `config` directory:

```php
<?php

return [

    /**
     * Public Key From Paystack Dashboard
     *
     */
    'publicKey' => env('PAYSTACK_PUBLIC_KEY'),

    /**
     * Secret Key From Paystack Dashboard
     *
     */
    'secretKey' => env('PAYSTACK_SECRET_KEY'),

    /**
     * Paystack Payment URL
     *
     */
    'paymentUrl' => env('PAYSTACK_PAYMENT_URL'),

    /**
     * Optional email address of the merchant
     *
     */
    'merchantEmail' => env('MERCHANT_EMAIL'),

];
```


## General payment flow

Though there are multiple ways to pay for an order, most payment gateways expect you to follow the following flow in your checkout process:

### 1. The customer is redirected to the payment provider
After the customer has gone through the checkout process and is ready to pay, the customer must be redirected to the site of the payment provider.

The redirection is accomplished by submitting a form with some hidden fields. The form must send a POST request to the site of the payment provider. The hidden fields minimally specify the amount that must be paid, the order id and a hash.

The hash is calculated using the hidden form fields and a non-public secret. The hash used by the payment provider to verify if the request is valid.


### 2. The customer pays on the site of the payment provider
The customer arrives on the site of the payment provider and gets to choose a payment method. All steps necessary to pay the order are taken care of by the payment provider.

### 3. The customer gets redirected back to your site
After having paid the order the customer is redirected back. In the redirection request to the shop site, some values are returned. The values are usually the order id, a payment result and a hash.

The hash is calculated out of some of the fields returned and a secret non-public value. This hash is used to verify if the request is valid and comes from the payment provider. It is paramount that this hash is thoroughly checked.


## Usage

Open your .env file and add your public key, secret key, merchant email, and payment URL like so:

```php
PAYSTACK_PUBLIC_KEY=xxxxxxxxxxxxx
PAYSTACK_SECRET_KEY=xxxxxxxxxxxxx
PAYSTACK_PAYMENT_URL=https://api.paystack.co
MERCHANT_EMAIL=merchant@gmail.com
```
*If you are using a hosting service like Heroku, ensure to add the above details to your configuration variables.*

Set up routes and controller methods like so:

Note: Make sure you have `/payment/callback` registered in Paystack Dashboard [https://dashboard.paystack.co/#/settings/developer](https://dashboard.paystack.co/#/settings/developer) like so:

![payment-callback](https://cloud.githubusercontent.com/assets/2946769/12746754/9bd383fc-c9a0-11e5-94f1-64433fc6a965.png)

```php
// Laravel 5.1.17 and above
Route::post('/pay', 'PaymentController@redirectToGateway')->name('pay');
```

OR

```php
Route::post('/pay', [
    'uses' => 'PaymentController@redirectToGateway',
    'as' => 'pay'
]);
```
OR

```php
// Laravel 8 & 9
Route::post('/pay', [App\Http\Controllers\PaymentController::class, 'redirectToGateway'])->name('pay');
```


```php
Route::get('/payment/callback', 'PaymentController@handleGatewayCallback');
```

OR

```php
// Laravel 5.0
Route::get('payment/callback', [
    'uses' => 'PaymentController@handleGatewayCallback'
]);
```

OR

```php
// Laravel 8 & 9
Route::get('/payment/callback', [App\Http\Controllers\PaymentController::class, 'handleGatewayCallback']);
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

use App\Http\Requests;
use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\Redirect;
use Paystack;

class PaymentController extends Controller
{

    /**
     * Redirect the User to Paystack Payment Page
     * @return Url
     */
    public function redirectToGateway()
    {
        try{
            return Paystack::getAuthorizationUrl()->redirectNow();
        }catch(\Exception $e) {
            return Redirect::back()->withMessage(['msg'=>'The paystack token has expired. Please refresh the page and try again.', 'type'=>'error']);
        }        
    }

    /**
     * Obtain Paystack payment information
     * @return void
     */
    public function handleGatewayCallback()
    {
        $paymentDetails = Paystack::getPaymentData();

        dd($paymentDetails);
        // Now you have the payment details,
        // you can store the authorization_code in your DB to allow for recurrent subscriptions
        // you can then redirect or do whatever you want
    }
}
```

```php
/**
 *  In the case where you need to pass the data from your 
 *  controller instead of a form
 *  Make sure to send:
 *  required: email, amount, reference, orderID(probably)
 *  optionally: currency, description, metadata
 *  e.g:
 *  
 */
$data = array(
        "amount" => 700 * 100,
        "reference" => '4g4g5485g8545jg8gj',
        "email" => 'user@mail.com',
        "currency" => "NGN",
        "orderID" => 23456,
    );

return Paystack::getAuthorizationUrl($data)->redirectNow();

```

Let me explain the fluent methods this package provides a bit here.
```php
/**
 *  This fluent method does all the dirty work of sending a POST request with the form data
 *  to the Paystack Api, then it gets the authorization URL and redirects the user to Paystack
 *  Payment Page. We've abstracted all of it, so you don't have to worry about that.
 *  Just eat your cookies while coding!
 */
Paystack::getAuthorizationUrl()->redirectNow();

/**
 * Alternatively, use the helper.
 */
paystack()->getAuthorizationUrl()->redirectNow();

/**
 * This fluent method does all the dirty work of verifying that the just-concluded transaction was actually valid,
 * It verifies the transaction reference with the Paystack Api and then grabs the data returned from Paystack.
 * In that data, we have a lot of good stuff, especially the `authorization_code` that you can save in your db
 * to allow for easy recurrent subscription.
 */
Paystack::getPaymentData();

/**
 * Alternatively, use the helper.
 */
paystack()->getPaymentData();

/**
 * This method gets all the customers who have performed transactions on your platform with Paystack
 * @returns array
 */
Paystack::getAllCustomers();

/**
 * Alternatively, use the helper.
 */
paystack()->getAllCustomers();


/**
 * This method gets all the plans that you have registered on Paystack
 * @returns array
 */
Paystack::getAllPlans();

/**
 * Alternatively, use the helper.
 */
paystack()->getAllPlans();


/**
 * This method gets all the transactions that have occurred
 * @returns array
 */
Paystack::getAllTransactions();

/**
 * Alternatively, use the helper.
 */
paystack()->getAllTransactions();

/**
 * This method generates a unique, super secure cryptographic hash token to use as a transaction reference
 * @returns string
 */
Paystack::genTranxRef();

/**
 * Alternatively, use the helper.
 */
paystack()->genTranxRef();


/**
* This method creates a subaccount to be used for split payments
* @return array
*/
Paystack::createSubAccount();

/**
 * Alternatively, use the helper.
 */
paystack()->createSubAccount();


/**
* This method fetches the details of a subaccount
* @return array
*/
Paystack::fetchSubAccount();

/**
 * Alternatively, use the helper.
 */
paystack()->fetchSubAccount();


/**
* This method lists the subaccounts associated with your Paystack account
* @return array
*/
Paystack::listSubAccounts();

/**
 * Alternatively, use the helper.
 */
paystack()->listSubAccounts();


/**
* This method updates a subaccount to be used for split payments
* @return array
*/
Paystack::updateSubAccount();

/**
 * Alternatively, use the helper.
 */
paystack()->updateSubAccount();
```

A sample form will look like so:

```php
<?php
// more details https://paystack.com/docs/payments/multi-split-payments/#dynamic-splits

$split = [
   "type" => "percentage",
   "currency" => "KES",
   "subaccounts" => [
    [ "subaccount" => "ACCT_li4p6kte2dolodo", "share" => 10 ],
    [ "subaccount" => "ACCT_li4p6kte2dolodo", "share" => 30 ],
   ],
   "bearer_type" => "all",
   "main_account_share" => 70
];
?>
```

```html
<form method="POST" action="{{ route('pay') }}" accept-charset="UTF-8" class="form-horizontal" role="form">
    <div class="row" style="margin-bottom:40px;">
        <div class="col-md-8 col-md-offset-2">
            <p>
                <div>
                    Lagos Eyo Print Tee Shirt
                    ₦ 2,950
                </div>
            </p>
            <input type="hidden" name="email" value="otemuyiwa@gmail.com"> {{-- required --}}
            <input type="hidden" name="orderID" value="345">
            <input type="hidden" name="amount" value="800"> {{-- required in kobo --}}
            <input type="hidden" name="quantity" value="3">
            <input type="hidden" name="currency" value="NGN">
            <input type="hidden" name="metadata" value="{{ json_encode($array = ['key_name' => 'value',]) }}" > {{-- For other necessary things you want to add to your payload. it is optional though --}}
            <input type="hidden" name="reference" value="{{ Paystack::genTranxRef() }}"> {{-- required --}}
            
            <input type="hidden" name="split_code" value="SPL_EgunGUnBeCareful"> {{-- to support transaction split. more details https://paystack.com/docs/payments/multi-split-payments/#using-transaction-splits-with-payments --}}
            <input type="hidden" name="split" value="{{ json_encode($split) }}"> {{-- to support dynamic transaction split. More details https://paystack.com/docs/payments/multi-split-payments/#dynamic-splits --}}
            {{ csrf_field() }} {{-- works only when using laravel 5.1, 5.2 --}}

            <input type="hidden" name="_token" value="{{ csrf_token() }}"> {{-- employ this in place of csrf_field only in laravel 5.0 --}}

            <p>
                <button class="btn btn-success btn-lg btn-block" type="submit" value="Pay Now!">
                    <i class="fa fa-plus-circle fa-lg"></i> Pay Now!
                </button>
            </p>
        </div>
    </div>
</form>
```

When clicking the submit button, the customer gets redirected to the Paystack site.

So now we've redirected the customer to Paystack. The customer did some actions there (hopefully he or she paid for the order) and now gets redirected back to our shop site.

Paystack will redirect the customer to the URL of the route that is specified in the Callback URL of the Web Hooks section on the Paystack dashboard.

We must validate if the redirect to our site is a valid request (we don't want impostors to wrongfully place non-paid order).

In the controller that handles the request coming from the payment provider, we have

`Paystack::getPaymentData()` - This function calls the verification methods and ensures it is a valid transaction otherwise, it throws an exception.

You can test with these details

```bash
Card Number: 4123450131001381
Expiry Date: any date in the future
CVV: 883
```

## Todo

* Charge Returning Customers
* Add Comprehensive Tests
* Implement the Transaction Dashboard to see all of the transactions in your Laravel app

## Contributing

Please feel free to fork this package and contribute by submitting a pull request to enhance the functionalities.

## Credits

- Original package: [unicodeveloper/laravel-paystack](https://github.com/unicodeveloper/laravel-paystack)
  by [Prosper Otemuyiwa](https://github.com/unicodeveloper) ([Follow on Twitter](https://twitter.com/unicodeveloper))!
- Updated and maintained by [Shohag](https://github.com/sajjadhossainshohag) ([Follow on Twitter](https://twitter.com/sajjadhshohag))!

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
