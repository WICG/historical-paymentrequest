# Better Web Payments

### What is this?

This is a proposal for a new web API that will allow merchants (i.e. websites selling physical or digital goods) to easily accept payments from multiple payment schemes with minimal integration.

### Why do we care?

Buying things on the web, particularly on mobile, is a frustrating experience for users. Every website has its own flow and its own validation rules, and most require users to manually type in the same set of information over and over again. Likewise, it is difficult and time consuming for developers to create good checkout flows that support various payment schemes.

We can fix this by allowing the User-Agent to act as an intermediary between three key parties in every transaction: The Merchant, the Buyer, and the Payment Instrument. Information necessary to process and confirm a transaction is passed between the Payment Instrument and the Merchant via the User-Agent with the Buyer confirming and authing as necessary across the flow.

In addition to better, more consistent user experiences, this also enables websites to take advantage of more secure payment schemes (e.g. tokenization and system-level authentication). This reduces liability for the merchant and helps protect sensitive user information.

### Goals

  - Allow the user-agent to act as intermediary between merchants, users, and payment instruments
  - Standardize the communication flow between a merchant, user-agent, and payment instrument
  - Easy installation and removal of payment instruments with User-Agents
  - Allow payment instruments to bring more secure payment methods to the web

### Non-goals

  - Not trying to create a new payment scheme
  - Not trying to integrate directly with payment processors

### Payment Request Lifecycle

1. Merchant requests payment with supported payment instruments
1. UA presents to user a list of payment instruments supported by merchant and installed by user as well as shipping options and address input/selection
1. User selects payment instrument of choice and, if shipping address is requested, selects or inputs a shipping address. 
1. Total transaction amount and other necessary data is passed to payment instrument for further processing.
1. Payment instrument returns back relevant data (e.g. credit card number, token, transaction ID) after successful authorization or transaction to UA
1. UA passes data back to merchant
2. Merchant finalizes and confirms transaction

A payment instrument can fundamentally take one of two actions: 1.) It can return back data necessary to finalize the transaction (but the transaction remains non-finalized); or 2.) It can complete the transaction and return back proof of the completed transaction.

An example of 1 would be a digital scheme like Apple Pay. Apple pay returns back data (e.g. a cryptogram) that the merchant must then submit to their processing partner to complete the transaction. An example of 2 might be a push payment like bitcoin, where the payment instrument is waiting to receive notification of successful transfer of funds.

### Delineation of Roles

**Merchant:** Responsible for calling `paymentRequest` on user action. Must be able to handle and parse response from supported payment instruments.

**User-Agent:** Acts as a selection agent for various user-installed payment instruments. Handles payment instrument selection and may handle address selection and input. Responsible for passing transaction details between merchant and payment instrument.

**Payment Instrument:** Fulfills a paymentRequest by passing back data a merchant can use to either finalize a transaction or prove a transaction has already been completed. 

### Basic Flow

To get started, a merchant creates a payment request. A payment request tells the browser what instruments or schemes are supported by the merchant, the amount of the transaction, and any other necessary scheme-specific information that may be necessary to process the transaction.

```js
var supportedInstruments = ["visa", "bitcoin", "bobpay.com"];

var details = {
    amount: 5500, //cents
    currencyCode: "USD",
    countryCode: "US",
    requestShipping: true,
    shippingOptions: shippingOptions,
    recurring: false
};

// Optional identities for schemes/instruments
var schemeData = {
    "bobpay.com": {
        merchantIdentifier: "XXXX",
        bobPaySpecificField: true
    },
    "bitcoin": {
        address: "XXXX"
    }
};

var pr = paymentRequest(supportedInstruments, details, schemeData);

pr.addEventListener("instrumentResponse", function(event) {
    console.log("Hurray, valid response from: ", event.instrumentName);
});

pr.addEventListener("error", function(event) {
    console.log("Uh oh: ", event.reason);
});
```

The `details` object is general to all payments. Any instrument-specific data is passed in as an optional object (`schemeData` in above example).

The User-Agent presents to the user the intersection of merchant-supported payment instruments and user-installed payment instruments. When the user selects a payment instrument, the User-Agent passes both `details` and instrument-specific data from `schemeData` off to selected intrument.

The `paymentRequest` object will implement [EventTarget](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget).

Restrictions:
  - Page calling `paymentRequest` must be HTTPS
  - Certain instruments may have their own restrictions

### Events

If `requestShipping` is enabled, the browser may provide address input ability for shipping. Shipping address changes may result in price or shipping option changes.

```js
pr.addEventListener("shippingAddressChange", function(event) {
    var newAddress = event.newAddress;
    console.log(newAddress.street, newAddress.city, newAddress.state, newAddress.zip, newAddress.country);
    // Return back new shipping options
    pr.updateShippingOptions(newShippingOpions);
});
```

```js
pr.addEventListener("instrumentResponse", function(event) {
    console.log("Hurray, valid response from: ", event.instrumentName, event.details);
});
```

```js
pr.addEventListener("error", function(event) {
    console.log("Oh no, so sad: ", event.reason);
});
```

### Methods

`updateShippingOptions`: Updates available shipping options. Typically called as a result of a shipping address change.

`pr.cancel`: Cancels the payment request.

### Errors

> ZK: How many of these are actually necessary to standardize? Could we get away with just NoAvailablePaymentInstrument and 'UserCanceledRequest'?

`NoAvailablePaymentInstruments`

`UserCanceledRequest`

`SubscriptionsNotSupported`

`AuthFailure`

### Shipping

Physical good transactions are the majority of e-commerce transactions on the mobile web. Merchants need a way to pass different shipping options into the paymentRequest. These options may be address-dependent, so if a user selects or adds a new address in the middle of a request, the merchant may need to update shipping options and prices.

Shipping options and address selection/input is handled by the User-Agent. The payment instrument is not privy to shipping information.

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

Payment instrument installation is platform-dependent, and the manifest can hint at the most appropriate form of installation for a given platform.

```js
"payment_instrument": {
    "url": "bobspayments.com"
    "label": "Bob\'s Payments",
    "description": "Bob\'s Payments will make all your payment dreams come true",
    "embeddable_url": "https://bobspayments.com/payment-request/",
    "related_applications": [
      {
        "platform": "play",
        "url": "https://play.google.com/store/apps/details?id=com.bobspayments.app1",
        "id": "com.example.app1"
      }, {
        "platform": "itunes",
        "url": "https://itunes.apple.com/app/bobspayments/id123456789",
      }]
}
```

This spec assumes some mechanism to securely verify a relationship between a website and a native app (e.g. [Android App Links](http://developer.android.com/preview/features/app-linking.html).

### Web & Native

User-Agents are free to determine the preferred payment instrument form for a given platform. For example, User-Agents on mobile devices may require that payment instruments be installed as native applications.

Payment instruments should in their manifest file, however, point to an embeddable_url that can communicate via `postMessage` that implements the `paymentRequest` message protocol. This allows all user-agents to support a payment instrument with basic iFrame communication protocols.

> ZK: How hard should we push to standardize something on desktop? Should we standardize that user-agents on desktops should support iFrames and postMessage? Or do we leave it up to the user-agents themselves to decide? I'm afraid if we leave it purely up to User-Agents, we are at risk for telling payment instruments that they have to write three different extensions for three different browsers. Thoughts?

### Nomenclature

"Payment instrument" can be a loaded term, and the line between instrument and scheme, particulary in this document, is a blurry one. For the sake of simplicity, I have referred to any thing the user selects and uses to pay for something as a 'payment instrument'.

This has, of course, some obvious weaknesses. The primary weakness being some some "instruments" actually have instruments within them. One example of this would be Apple Pay. A user might say, "I want to pay with Apple Pay," but in reality, they are selecting a payment instrument within Apple Pay to pay with (e.g. A Bank of America Visa Card).

In the future, we may want to better align these terms with the terms defined in the [Web Payments Working Group Charter](http://www.w3.org/2015/06/payments-wg-charter.html).

### Open Questions

  - How can we let merchants know if a payment instrument is available before calling paymentRequest()?
  - Should we standardize the response for "general" payment instruments (e.g. standard credit/debit cards?)
  - Subscriptions - only select payment instruments with recurring billing
  - Desktop. How? iFrames? How much do we standardize?
  - Do we support general schemes? e.g. ACH that anyone can work with or only Dwolla?
