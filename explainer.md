# Better Web Payments

### Payments are hard

Buying things on the web, particularly on mobile, is a frustrating experience for users. Every website has its own flow, its own validation rules, and most require users to manually type in the same set of information over and over again. Likewise, it is difficult and time consuming for developers to create good checkout flows that support various payment schemes.

We can fix this by allowing the User-Agent to act as a proxy between the three key parties in every transaction: The Merchant, the Buyer, and the Payment Instrument. Information necessary to process and confirm a transaction is passed between the Payment Instrument and the Merchant via the User-Agent with the User confirming and authing as necessary across the flow.

In addition to better, more consistent user experiences across all sites, this also enables websites to take advantage more secure payment schemes (e.g. tokenization and system-level authentication). This reduces liability for the merchant and helps protect sensitive user information.

### Goals

  - Standardize the communication flow between a merchant (website), user-agent, and payment instrument
  - User-agent acts as proxy between merchant and user-installed payment instruments
  - Easy installation and removal of payment instruments for users
  - Bring more secure payment methods to the web

### Payment Request Lifecycle

1. Website requests payment with supported payment instruments
1. UA presents to user a list of payment instruments supported by merchant and installed by user as well as shipping options
1. User selects payment instrument of choice and, if shipping address is requested, selects or inputs a shipping address. 
1. Total transaction amount and other necessary data is passed to payment instrument for further processing.
1. Payment instrument returns back relevant data (e.g. credit card number, token, transaction ID) after successful authorization or transaction to UA
1. Data is passed through UA back to merchant

A payment instrument can fundamentally take one of two actions: 1.) It can return back data necessary to finalize the transaction (but the transaction remains non-finalized); or 2.) It can complete the transaction and return back proof of the completed transaction

An example of 1 would be a digital scheme like Apple Pay. Apple pay returns back data (e.g. a cryptogram) that the merchant must then submit to their processing partner to complete the transaction. An example of 2 might be a push payment like bitcoin, where the payment instrument is waiting to receive notification of successful transfer of funds.

### Delineation of Roles

**Website:** Responsible for calling `paymentRequest` on user action. Must be able to handle and parse response from supported payment instruments.

**User-Agent:** Acts as a selection agent for various user-installed payment instruments. Handles payment instrument selection and may handle address selection and input.

**Payment Instrument:** Fulfills a paymentRequest by passing back data a website can use to either finalize a transaction or prove a transaction has already been completed. 

### Basic Flow

To get started, a merchant creates a payment request. A payment request tells the browser what instruments or schemes are supported by the website, the amount of the transaction, and any other necessary scheme-specific information that may be necessary to process the transaction.

```js
var supportedInstruments = ['visa', 'bitcoin', 'bobpay'];

var details = {
    amount: 5500, //cents
    currencyCode: 'USD',
    countryCode: 'US',
    requestShipping: true,
    shippingOptions: shippingOptions,
    recurring: false
};

// Optional identities for schemes/instruments
var schemeData = {
    'bobpay': {
        merchantIdentifier: 'XXXX',
        bobPaySpecifcField: true
    },
    'bitcoin': {
        address: 'XXXX'
    }
};

var pr = paymentRequest(supportedInstruments, details, schemeData);

pr.addEventListener('instrumentResponse', function(event) {
    console.log('Hurray, valid response from: ', event.instrumentName);
});

pr.addEventListener('error', function(event) {
    console.log('Uh oh: ', event.reason);
});
```

The `details` object is general to all payments. Any instrument-specific data is passed in as an optional object.

The User-Agent presents to the user the intersection of merchant-supported payment instruments and user-installed payment instruments. When the user selects a payment instrument, the User-Agent passes both `details` and instrument-specific data from `schemeData` off to selected intrument.

The `paymentRequest` object will implement [EventTarget](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget).

> ZK: Are strings the best way to handle payment instrument selection? How do we stop bad people from registering themselves as supported instruments for schemes they don't really support (e.g. bobpay pretending to support fredpay)?

> ZK: I started off with paymentRequest() returning a promise, but given that we need to support shipping addresses, which mean communicating back with the merchant, the promise no longer seems like the [right move](https://www.w3.org/2001/tag/doc/promises-guide#one-time-events)

Restrictions:
  - Page calling `paymentRequest` must be HTTPS
  - Certain instruments may have their own restrictions

### Events

If `requestShipping` is enabled, the browser may provide address input ability for shipping. Shipping address changes may result in price or shipping option changes.

```js
pr.addEventListener('shippingAddressChange', function(event) {
    var newAddress = event.newAddress;
    console.log(newAddress.street, newAddress.city, newAddress.state, newAddress.zip, newAddress.country);
    // Return back new shipping options
    pr.updateShippingOptions(newShippingOpions);
});
```

```js
pr.addEventListener('instrumentResponse', function(event) {
    console.log('Hurray, valid response from: ', event.instrumentName, event.details);
});
```

```js
pr.addEventListener('error', function(event) {
    console.log('Oh no, so sad: ', event.reason);
});
```

### Methods

`updateShippingOptions`: Updates available shipping options. Typically called as a result of a shipping address change.

`pr.cancel`: Cancels the payment request.

### Errors

> ZK: How many of these are actually necessary to standardize? Could we get away with just NoAvailablePaymentInstrument and 'UserCanceledRequest'?

> ZK: What's the cleanest way for payment instruments to send back custom error messages? 

`NoAvailablePaymentInstruments`

`UserCanceledRequest`

`SubscriptionsNotSupported`

`AuthFailure`

`InadequateAccountBalance`

### Shipping

Physical good transactions are the majority of e-commerce transactions on the mobile web. Merchants need a way to pass different shipping options into the paymentRequest. These options may be address-dependent, so if a user selects or adds a new address in the middle of a request, the merchant may need to update shipping options and prices.

Shipping options and address selection is handled by the UA. The payment instrument has no notion of where the thing being purchased is shipped to.

```js
var shippingOptions = [
    {
        label: "Standard shipping",
        identifier: "standardShipping",
        description: "Arrives in 6-8 weeks",
        amount: 500,
    },
    {
        label: "2-day shipping",
        identifier: "twoDayShipping",
        description: "Arrives in 2 business days",
        amount: 1000
    }
];
```

### Registering Payment Instruments

Web site owners can declare they have a supported payment instrument via their [Manifest](https://developers.google.com/web/updates/2014/11/Support-for-installable-web-apps-with-webapp-manifest-in-chrome-38-for-Android?hl=en). User-Agents, upon seeing this declaration, can present to the user the option of installing the payment instrument. 

Payent instrument installation is platform-dependent (i.e. mobile may work differntly than desktop).

```js
paymentInstrument: {
    instrumentName: "bobsPayments"
    label: 'Bob\'s Payments',
    description: 'Bob\'s Payments will make all your payment dreams come true',
    embeddableURL: 'https://bobspayments.com/payment-request/',
    appleAppStore: '...',
    googlePlayStore: '...'
}
```

### Payment Instruments or Schemes?

Nomenclature is hard. Payment instruments can be a loaded term, and the line between instrument and scheme, particulary in this document, is a blurry one. For the sake of simplicity, I have referred to any thing the user chooses to pay for something as a 'payment instrument'.

But the complexity doesn't quite end there. In reality, some payment instruments will only be supported by a single entity while others can technically be supported by multiple entities. As an example, take your standard credit or debit card. A user could have multiple "payment instruments" installed that can return back a valid credit/debit card. Examples of these might include OnePassword and Chrome's Autofill. Both applications would want to register as supported payment instruments for your standard schemes (e.g. Visa, MC, Discover, etc). Other payment instruments (e.g. PayPal), would only be supported by a single entity (e.g. PayPal.com) and no one else should be able to register under that instrument name. What is the right labeling? Should we standardize particular intrument types (e.g. credit cards) and expected responses? Should the UA have to know what are "common" instrument as opposed to private instruments?

### Open Questions

  - Should we standardize the response for "general" payment instruments (e.g. standard credit/debit cards?)
  - Subscriptions - only select payment instruments with recurring billing
  - Desktop. How? iFrames? How much do we standardize?
  - Do we support general schemes? e.g. ACH that anyone can work with or only Dwolla?
