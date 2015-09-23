# Better Web Payments

### What is this?

This is a proposal for a new web API that will allow merchants (i.e. websites selling physical or digital goods) to easily accept payments from multiple payment schemes and instruments with minimal integration.

This proposal attempts to align itself with the chartered scope of the proposed Web Payment Interest Group, and the expectation is to offer this document as input to the discussions of that Working Group once it has launched.

### Why do we care?

Buying things on the web, particularly on mobile, is a frustrating experience for users. Every website has its own flow and its own validation rules, and most require users to manually type in the same set of information over and over again. Likewise, it is difficult and time consuming for developers to create good checkout flows that support various payment schemes.

We can fix this by allowing the User-Agent (e.g. browsers) to act as an **intermediary** between the three key parties in every transaction: The Merchant (e.g. an online web store), the Buyer (e.g. the user buying from the online web store), and the Payment Instrument (e.g. credit card, Bitcoin, Apple Pay). Information necessary to process and confirm a transaction is passed between the Payment Instrument and the Merchant via the User-Agent with the Buyer confirming and authing as necessary across the flow.

In addition to better, more consistent user experiences, this also enables websites to take advantage of more secure payment schemes (e.g. tokenization and system-level authentication) that are not possible with standard JavaScript libraries. This has the potential to reduce liability for the merchant and helps protect sensitive user information.

### Goals

  - Allow the user-agent to act as intermediary between merchants, users, and payment instruments
  - Standardize (to the extent that it makes sense) the communication flow between a merchant, user-agent, and payment instrument
  - Easy installation and removal of payment instruments with User-Agents
  - Allow payment instruments to bring more secure payment methods to the web

### Non-goals

  - Not trying to create a new payment scheme
  - Not trying to integrate directly with payment processors

### Payment Request Lifecycle

1. Merchant requests payment with supported payment instrument(s)
1. UA presents to user a list of payment instruments accepted by merchant and installed by user
1. User selects payment instrument of choice and, if shipping address is requested, selects or inputs a shipping address. 
1. Total transaction amount and other necessary data is passed to payment instrument for further processing.
1. Payment instrument returns back relevant data (e.g. credit card number, token, transaction ID) after successful authorization or transaction to UA
1. UA passes data back to merchant
1. Merchant independently finalizes and confirms transaction

![paymentRequest Flow Diagram](https://github.com/zkoch/paymentrequest/blob/master/flow_diagram.png)

A payment instrument can fundamentally take one of two actions: 1.) It can return back data necessary to finalize the transaction (but the transaction remains non-finalized); or 2.) It can complete the transaction and return back proof of the completed transaction.

An example of 1 would be a digital scheme like Apple Pay. Apple pay returns back data (e.g. a cryptogram) that the merchant must then submit to their processing partner to complete the transaction. An example of 2 might be a push payment like Bitcoin, where the payment instrument is waiting to receive notification of successful transfer of funds.

**From a user perspective:**

> Alice has added a few things to her shopping cart and is ready to check out. She clicks the buy button, which causes the website to request payment via the browser paymentRequest API. The website tells the API which payment instruments it accepts and the browser asks the user to pick which supported instrument Alice wants to use to pay (e.g. Visa or Bitcoin). If the merchant requests shipping information, Alice also selects a shipping address from locations already stored in the browser or inputs a new one. She submits all of this to the browser which then sends the transaction details (e.g. total amount) to her selected payment instrument. The payment instrument then authorizes or confirms the transaction and passes back details to the UA, which then passes them back to the merchant's website. The website then confirms to Alice that her order has been placed. Sometimes Alice might need to give the website or payment instrument more information in order to complete the process, but in nearly every case this flow will be faster and easier than today's checkout systems.

### Delineation of Roles

**Merchant:** Responsible for calling `paymentRequest` on a user action. Must be able to handle and parse response from supported payment instruments.

**User-Agent:** Acts as a selection agent for various user-installed payment instruments. Handles payment instrument selection and may handle address selection and input if requested by merchant. Responsible for passing transaction details between merchant and payment instrument.

**Payment Instrument:** Fulfills a `paymentRequest` and passes back data a merchant can use to either finalize a transaction or prove a transaction has already been completed. 

### Basic Flow

To get started, a merchant creates a payment request. A payment request tells the browser what instruments or schemes are accepted by the merchant, the amount of the transaction, and any other necessary scheme-specific information that may be necessary to process the transaction. The standard message passing format between entities is JSON.

```js
var supportedInstruments = ["visa", "bitcoin", "bobpay.com"];

var details = {
    "amount": 5500, //cents
    "currencyCode": "USD",
    "countryCode": "US",
    "requestShipping": true,
    "recurringCharge": false
};

// Optional identities for schemes/instruments
var schemeData = {
    "bobpay.com": {
        "merchantIdentifier": "XXXX",
        "bobPaySpecificField": true
    },
    "bitcoin": {
        "address": "XXXX"
    }
};

var promise = paymentRequest(supportedInstruments, details, schemeData);

promise.then(function(requester) {
    requester.addEventListener("shippingAddressChange", function () {
        var newAddress = requester.userAddress;
        // Return back new shipping options
        requester.updateShippingOptions([...]);
    });
    return requester.instrumentResponse; //always necessary
})
.then(function(instrumentResponse) {
    console.log("hurray, valid response from ", instrumentResponse.instrumentName, instrumentResponse.details)''
})
.catch(function(err) {
	// Typically no available payment instruments or error on instrument side
   console.error("Uh oh, something bad happened", err.message);
});
```

The `details` object is general to all payments. Any instrument-specific data is passed in as an optional object (`schemeData` in above example).

The User-Agent presents to the user the intersection of merchant-supported payment instruments and user-installed payment instruments. When the user selects a payment instrument, the User-Agent passes both `details` and instrument-specific data from `schemeData` off to selected intrument.

Restrictions:
  - Page calling `paymentRequest` must be HTTPS
  - Certain instruments may have their own restrictions

### Events

**`shippingAddressChange`**:

When a user selects or inputs a new shipping address, the website may want to listen for this event in order to update available shipping options. 

### Methods

Both of these methods are available on the returned object from the initial `paymentRequest` call.

`requester.updateShippingOptions`: Updates available shipping options. Typically called as a result of a shipping address change.

`requester.cancel`: Cancels the payment request.

### Errors

`NoAvailablePaymentInstruments`

`UserCanceledRequest`

`NoShippingOptionsProvided`

### Shipping

Physical good transactions are the majority of e-commerce transactions on the mobile web. Merchants need a way to pass different shipping options into the paymentRequest. These options may be address-dependent, so if a user selects or adds a new address in the middle of a request, the merchant may need to update shipping options and prices. Default shipping options can optionally be passed into the initial `paymentRequest` as `defaultShippingOptions` in the `details` object.

Shipping options and address selection/input is handled by the User-Agent. The payment instrument is not privy to shipping information.

If `requestShipping` is enabled, but the merchant hasn't supplied shipping options at the completion of the flow, the UA should throw a `NoShippingOptionsProvided` error.

> ZK: Should the UA be able to limit the amount of time a website has to respond with updated shipping options after a shippingAddressChange event? This would be to prevent a hanging UI since new shipping options might be necessary to complete a transction. A .isExpired() function could be in the requester object.

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

requester.addEventListener("shippingAddressChange", function() {
    var newAddress = requester.newAddress;
    console.log(newAddress.street, newAddress.city, newAddress.state, newAddress.zip, newAddress.country);
    // Return back new shipping options
	 requester.updateShippingOptions(shippingOpions);
});
```

### Standardizing Common Payment Instruments

The overwhelming majority of transactions currently taking place on the web use a credit or debit card. Since it's possible for multiple payment instruments to return back credit or debit card numbers, we should standardize the `instrumentResponse` object for these transactions. The browser should verify a valid card numbers with the [Luhn check](https://en.wikipedia.org/wiki/Luhn_algorithm).

**Credit or debit card response:**

```js
{
    "instrumentName": "Visa",
    "details": {
        "cardNumber": "1234123412341234",
        "nameOnCard": "Bob J. Paymentman",
        "expMonth": "12",
        "expYear": "2016",
        "cvv2": "123"
    }
}
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

> ZK: How hard should we push to standardize something on desktop? Should we standardize that user-agents on desktops should support iFrames and postMessage? Or do we leave it up to the user-agents themselves to decide? I'm afraid if we leave it purely up to User-Agents, we are at risk for telling payment instruments that they have to write three different extensions for three different browsers. Thoughts?

### Nomenclature

"Payment instrument" can be a loaded term, and the line between instrument and scheme, particulary in this document, is a blurry one. For the sake of simplicity, I have referred to any thing the user selects and uses to pay for something as a 'payment instrument'.

This has, of course, some obvious weaknesses. The primary weakness being some some "instruments" actually have instruments within them. One example of this would be Apple Pay. A user might say, "I want to pay with Apple Pay," but in reality, they are selecting a payment instrument within Apple Pay to pay with (e.g. A Bank of America Visa Card).

In the future, we may want to better align these terms with the terms defined in the [Web Payments Working Group Charter](http://www.w3.org/2015/06/payments-wg-charter.html).

### Open Questions

  - How can we let merchants know if a payment instrument is available before calling paymentRequest()?
  - Subscriptions - only select payment instruments with recurring billing
  - Desktop. How? iFrames? How much do we standardize?
  - Do we support general schemes? e.g. ACH that anyone can work with or only Dwolla?