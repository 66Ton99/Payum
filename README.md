Payum [![Build Status](https://travis-ci.org/Payum/Payum.png?branch=master)](https://travis-ci.org/Payum/Payum) [![Total Downloads](https://poser.pugx.org/payum/payum/d/total.png)](https://packagist.org/packages/payum/payum) [![Latest Stable Version](https://poser.pugx.org/payum/payum/version.png)](https://packagist.org/packages/payum/payum)
=====

It is a full-fledged payments library.

#### Supported payments

* [Paypal Express Checkout Nvp](https://github.com/Payum/PaypalExpressCheckoutNvp)
* [Paypal Pro Checkout Nvp](https://github.com/Payum/PaypalProCheckoutNvp)
* [Authorize.Net AIM](https://github.com/Payum/AuthorizeNetAim)
* [Be2Bill](https://github.com/Payum/Be2Bill)
* All gateways supported by [omnipay](https://github.com/adrianmacneil/omnipay) lib via [bridge](https://github.com/Payum/OmnipayBridge).

#### Frameworks supported.

* Symfony2 [bundle][payum-bundle] and sandbox: [code][sandbox-code], [online][sandbox-online].
* Zend2 module to be implemented.

#### The architecture

_**Note**: The code snippets presented below are only for demonstration purposes (a pseudo code). Its goal is to illustrate general approach to various tasks. To see real live examples follow the links provided when appropriate._

_**Note**: If you'd like to see real world examples we have a sandbox: [online][sandbox-online], [code][sandbox-code]._

In general, you have to create a _[request][base-request]_ and have to have an _[action][action-interface]_ which knows what to do with such request. A _[payment][payment-interface]_ contains sets of actions and can execute a request. So, payment is the place where a request and an action meet each other.

```php
<?php
$payment = new Payment;
$payment->addAction(new CaptureAction));

//CaptureAction does its job.
$payment->execute($capture = new CaptureRequest(array(
    'amount' => 100,
    'currency' => 'USD'
));

var_export($capture->getModel());
```

```php
<?php
class CaptureAction implements ActionInterface
{
    public function execute($request)
    {
       $model = $request->getModel();
       
       //capture payment logic here
       
       $model['status'] = 'success';
       $model['transaction_id'] = 'an_id';
    }
    
    public function supports($request)
    {
        return $request instanceof CaptureRequest;
    }
}
```

Here's a real world [example][capture-controller].

That's a big picture. Now let's talk about the details:

* An action does not want to do all the job alone, so it delegates some responsibilities to other actions. In order to achieve this the action must be a _payment aware_ action. Only then, it can create a sub request and pass it to the payment.

    ```php
    <?php
    class FooAction extends PaymentAwareAction
    {
        public function execute($request)
        {
            //do its jobs

            // delegate some job to bar action.
            $this->payment->execute(new BarRequest);
        }
    }
    ```

    See paypal [capture action][paypal-capture-action].

* What about redirects? Some payments like paypal requires authorization on their side. The payum can handle such cases and for that we use something called _[interactive requests][base-interactive-request]_. It is a special request object, which extends an exception. You can throw interactive redirect request at any time and catch it at a top level.

    ```php
    <?php
    class FooAction implements ActionInterface
    {
        public function execute($request)
        {
            throw new RedirectUrlInteractiveRequest('http://example.com/auth');
        }
    }
    ```
    ```php
    <?php
    try {
        $payment->addAction(new FooAction);

        $payment->execute(new FooRequest);
    } catch (RedirectUrlInteractiveRequest $redirectUrlInteractiveRequest) {
        header( 'Location: '.$redirectUrlInteractiveRequest->getUrl());
        exit;
    }
    ```

    See paypal [authorize token action][paypal-authorize-token-action].

* Good status handling is very important. Statuses must not be hard coded and should be easy to reuse, hence we use an _[interface][status-request-interface]_ to hanle this. [Status request][status-request] is provided by default by our library, however you are free to use your own and you can do so by implementing status interface.

    ```php
    <?php
    class FooAction implements ActionInterface
    {
        public function execute($request)
        {
            if ('success condition') {
               $request->markSuccess();
            } else if ('pending condition') {
               $request->markPending();
            } else {
               $request->markUnknown();
            }
        }

        public function supports($request)
        {
            return $request instanceof StatusRequestInterface;
        }
    }
    ```
    ```php
    <?php
    
    $payment->addAction(new FooAction);

    $payment->execute($status = new BinaryMaskStatusRequest);

    $status->isSuccess();
    $status->isPending();

    // or

    $status->getStatus();
    ```

    The status logic could be a bit [complicated][paypal-status-action] or pretty [simple][authorize-status-action]. 

* There must be a way to extend the payment with custom logic. _[Extension][extension-interface]_ to the rescue. Let's look at the example below. Imagine you want to check permissions before user can capture the payment:

    ```php
    <?php
    class PermissionExtension implements ExtensionInterface
    {
        public function onPreExecute($request)
        {
            if (false == in_array('ROLE_CUSTOMER', $request->getModel()->getRoles())) {
                throw new Exception('The user does not have required roles.');
            }

            // congrats, user have enough rights.
        }
    }
    ```
    ```php
    <?php
    $payment->addExtension(new PermissionExtension);
    
    // here the place the exception may be thrown.
    $payment->execute(new FooRequest);
    ```
    
    The [storage extension][storage-extension-interface] may be a good extension example.

* Before you are redirected to a gateway side, you may want to store data somewhere, right? We take care of that too. This is handled by _[storage][storage-interface]_ and its _[storage extension][storage-extension-interface]_ for payment. The extension can solve two tasks. First it can save a model after the request is processed. Second, it could find a model by its id before request is processed. Currently [Doctrine][doctrine-storage] and [filesystem][filesystem-storage] (use it for tests only!) storages are supported.

    ```php
    <?php
    $storage = new FooStorage;

    $payment = new Payment;
    $payment->addExtension(new StorageExtension($storage));
    ```
* The payment API can have different versions? No problem! A payment may contain a set of APIs. When _API aware action_ is added to a payment it tries to set an API one by one to the action until the action accepts one.

    ```php
    <?php
    class FooAction implements ActionInterface, ApiAwareInterface
    {
        public function setApi($api)
        {
            if (false == $api instanceof FirstApi) {
                throw new UnsupportedApiException('Not supported.');
            }
        
            $this->api = $api;
        }
    }

    class BarAction implements ActionInterface, ApiAwareInterface
    {
        public function setApi($api)
        {
            if (false == $api instanceof SecondApi) {
                throw new UnsupportedApiException('Not supported.');
            }
        
            $this->api = $api;
        }
    }
    ```
    ```php
    <?php
    $payment = new Payment;
    $payment->addApi(new FirstApi);
    $payment->addApi(new SecondApi);

    //here the ApiVersionOne will be injected to FooAction
    $payment->addAction(new FooAction);

    //here the ApiVersionTwo will be injected to BarAction
    $payment->addAction(new BarAction);
    ```

    See authorize.net [capture action][authorize-capture-action].

As a result of such architecture we have decoupled, easy to extend and reusable library. For example, you can add your domain specific actions or a logger extension. Thanks to its flexibility any task could be achieved.

### The bundle architecture

_**Note:** There is a [doc][bundle-doc] on how to setup and use payum bundle with supported payment gateways._

The bundle allows you easy configure payments, add storages, custom actions/extensions/apis. Nothing is hardcoded: all payments and storages are added via _factories_ ([payment factories][payment-factories], [storage factories][storage-factories]) in the bundle [build method][payum-bundle]. You can add your payment this way too!

Also, it provides a nice secured [capture controller][capture-controller]. It's extremely reusable. Check the [sandbox][sandbox-online] ([code][sandbox-code]) for more details.

The bundle supports [omnipay][omnipay] gateways (up to 25) out of the box. They could be configured the same way as native payments. The capture controller works [here][omnipay-example] too.

### How to capture?

```php
<?php
//Source: Payum\Examples\ReadmeTest::bigPicture()
use Payum\Examples\Action\CaptureAction;
use Payum\Examples\Action\StatusAction;
use Payum\Request\CaptureRequest;
use Payum\Payment;

//Populate payment with actions.
$payment = new Payment;
$payment->addAction(new CaptureAction());

//Create request and model. It could be anything supported by an action.
$captureRequest = new CaptureRequest(array(
    'amount' => 10,
    'currency' => 'EUR'
));

//Execute request
$payment->execute($captureRequest);

echo 'We are done!';
echo 'We are done!';
```

### How to manage authorize redirect?

```php
<?php
//Source: Payum\Examples\ReadmeTest::interactiveRequests()
use Payum\Examples\Request\AuthorizeRequest;
use Payum\Examples\Action\AuthorizeAction;
use Payum\Request\CaptureRequest;
use Payum\Request\RedirectUrlInteractiveRequest;
use Payum\Payment;

$payment = new Payment;
$payment->addAction(new AuthorizeAction());

$request = new AuthorizeRequest($model);

if ($interactiveRequest = $payment->execute($request, $catchInteractive = true)) {    
    if ($interactiveRequest instanceof RedirectUrlInteractiveRequest) {
        echo 'User must be redirected to '.$interactiveRequest->getUrl();
    }

    throw $interactiveRequest;
}
```

### How to check payment status

```php
<?php
//Source: Payum\Examples\ReadmeTest::gettingRequestStatus()
use Payum\Examples\Action\StatusAction;
use Payum\Request\BinaryMaskStatusRequest;
use Payum\Payment;

//Populate payment with actions.
$payment = new Payment;
$payment->addAction(new StatusAction());

$statusRequest = new BinaryMaskStatusRequest($model);
$payment->execute($statusRequest);

//Or there is a status which require our attention.
if ($statusRequest->isSuccess()) {
    echo 'We are done!';
} 

echo 'Uhh something wrong. Check other possible statuses!';
echo 'Uhh something wrong. Check other possible statuses!';
```

### How to persist payment details?

```php
<?php
//Source: Payum\Examples\ReadmeTest::persistPaymentDetails()
use Payum\Payment;
use Payum\Storage\FilesystemStorage;
use Payum\Extension\StorageExtension;

$storage = new FilesystemStorage('path_to_storage_dir', 'YourModelClass', 'idProperty');

$payment = new Payment;
$payment->addExtension(new StorageExtension($storage));

//do capture for example.
```
What's inside? 

* The extension will try to find model on `onPreExecute` if an id given.
* Second, It saves the model after execute, on `onInteractiveRequest` and `postRequestExecute`.

### Like it? Spread the world!

You can star the lib on [github](https://github.com/Payum/Payum) or [packagist](https://packagist.org/packages/payum/payum). You may also drop a message on Twitter.  

### Need support?

If you are having general issues with [payum](https://github.com/Payum/Payum), we suggest posting your issue on [stackoverflow](http://stackoverflow.com/). Feel free to ping @maksim_ka2 on Twitter if you can't find a solution.

If you believe you have found a bug, please report it using the [GitHub issue tracker](https://github.com/Payum/Payum/issues), or better yet, fork the library and submit a pull request.

### License

Payum is released under the MIT License. For more information, see [License](LICENSE).

[sandbox-online]: http://sandbox.payum.forma-dev.com
[sandbox-code]: https://github.com/Payum/PayumBundleSandbox
[base-request]: https://github.com/Payum/Payum/blob/master/src/Payum/Request/BaseModelRequest.php
[status-request-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/Request/StatusRequestInterface.php
[status-request]: https://github.com/Payum/Payum/blob/master/src/Payum/Request/BinaryMaskStatusRequest.php
[base-interactive-request]: https://github.com/Payum/Payum/blob/master/src/Payum/Request/BaseInteractiveRequest.php
[action-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/Action/ActionInterface.php
[extension-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/Extension/ExtensionInterface.php
[storage-extension-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/Extension/StorageExtension.php
[storage-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/Storage/StorageInterface.php
[doctrine-storage]: https://github.com/Payum/Payum/blob/master/src/Payum/Bridge/Doctrine/Storage/DoctrineStorage.php
[filesystem-storage]: https://github.com/Payum/Payum/blob/master/src/Payum/Storage/FilesystemStorage.php
[payment-interface]: https://github.com/Payum/Payum/blob/master/src/Payum/PaymentInterface.php
[capture-controller]: https://github.com/Payum/PayumBundle/blob/master/Controller/CaptureController.php
[paypal-capture-action]: https://github.com/Payum/PaypalExpressCheckoutNvp/blob/master/src/Payum/Paypal/ExpressCheckout/Nvp/Action/CaptureAction.php
[paypal-authorize-token-action]: https://github.com/Payum/PaypalExpressCheckoutNvp/blob/master/src/Payum/Paypal/ExpressCheckout/Nvp/Action/Api/AuthorizeTokenAction.php
[paypal-status-action]: https://github.com/Payum/PaypalExpressCheckoutNvp/blob/master/src/Payum/Paypal/ExpressCheckout/Nvp/Action/PaymentDetailsStatusAction.php
[authorize-capture-action]: https://github.com/Payum/AuthorizeNetAim/blob/master/src/Payum/AuthorizeNet/Aim/Action/CaptureAction.php
[authorize-status-action]: https://github.com/Payum/AuthorizeNetAim/blob/master/src/Payum/AuthorizeNet/Aim/Action/StatusAction.php
[omnipay]: https://github.com/adrianmacneil/omnipay
[omnipay-example]: https://github.com/Payum/PayumBundleSandbox/blob/master/src/Acme/PaymentBundle/Controller/SimplePurchasePaypalExpressViaOmnipayController.php
[bundle-doc]: https://github.com/Payum/PayumBundle/blob/master/Resources/doc/index.md
[payment-factories]: https://github.com/Payum/PayumBundle/tree/master/DependencyInjection/Factory/Payment
[storage-factories]: https://github.com/Payum/PayumBundle/tree/master/DependencyInjection/Factory/Storage
[payum-bundle]: https://github.com/Payum/PayumBundle/blob/master/PayumBundle.php
