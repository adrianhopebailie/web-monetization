# Web Monetization Explainer

Web Monetization is a proposed API standard that allows websites to request a stream of very small payments from a user.

This provides a framework for new revenue models for websites and web-based services.

In exchange for payments from the user websites can provide the user with a "premium" experience such as exclusive content or by removing advertising or even the need to login to access content or services.

## Goals

Provide websites with a way to collect multiple small payments from users in exchange for consuming the content and/or services on the website.

The experience must be frictionless for users. It must allow users to pre-approve payments in aggregate or delegate the authorization of the individual small payments to a third-party (a Web Monetization provider) that interacts with the website without the need for user interaction.

It should not be possible for websites to identify users on the basis of the payments they make.

It should not be possible for the user's Web Monetization provider to identify which websites the user is visiting.

## Non-goals

Online purchases. Web Monetization is intended to enable very small payments. This distinction is important because very small payments can be performed with different levels of user consent to larger payments such as those used in traditional e-commerce. 


## Overview of the Flow

1. Users enroll with one or more **Web Monetization providers (WM provider)**, responsible for making payments to websites on the user's behalf.

2. When the user visits a website that uses Web Monetization the website provides the browser with an address at which they wish to receive payments.

3. The browser connects the user's WM provider with the website's receiving service providing a random, unique identifier for the payment stream allowing the website to correlate the payments coming in at the receiver to the browser session with the user.

4. The WM provider notifies the browser each time it sends a payment which the browser raises as an event that can be consumed by the website via the API.

## Why is a standard required?

There are many services attempting to provide alternative means to monetize the Web and generate revenue for content creators and service providers without selling ads. 

However, most of these require that the user and the creator/producer/service provider join a common network that offers to facilitate the transactions between users and these services.

The result is a fragmented Web of closed content and service silos rather than the global and open Web we desire.

With Web Monetization, WM providers compete for users (as customers) not by trying to build a bigger network of content partners, but by delivering a better service.

Further, by decoupling the provider and the service, using the browser as an intermediary, the privacy of users is protected and payments can't be used to track a user across sites.

## Design Decisions

This proposal is modelled on a working deployment (setup by the WM Provider, [Coil](https://coil.com)) that uses a browser extension to provide the necessary browser-side functionality, however there are various design decisions that may be worth discussing further as a community.

By bringing this work to the WICG our goal is to get input from multiple WM Providers and implementors to refine the design and produce a W3C standards-track specification.

### Declaritive vs Imperative?

The current proposal is for a hybrid declarative and imperative API whereby websites declare their ability to accept micropayments using a `<meta>` tag in the page header and then access the global `monetization` object on the DOM to track incoming payment events and respond to these.

### Payment Request and Payment Handler APIs

The Web Payments WG has designed two APIs that follow a similar pattern to Web Monetization but for a slightly different use case. 

The Payment Request API is an imperative API that websites can use to request a single discreet payment.

This is designed to always prompt the user for authorization as part of the flow as it is designed for payment sizes where this is necessary. However, nothing prevents this API also supporting a non-interactive flow that supports Web Monetization use cases.

Further, the Payment Handler API aligns well with the model anticipated for Web Monetization providers. A provider might manifest as a specialized Payment Handler capable of returning not just a `PaymentRequestResponse` but also a handle to a stream of micropayments.

### Streams

In-keeping with the trend toward streaming APIs the API surface could be updated to implement the stream API.

## Concepts

Web Monetization depends on two important technologies/concepts that enable open and interoperable payments between providers and websites for very small amounts.

### Interledger

The Interledger protocol is a protocol for inter-networking existing payments networks. It operates as an open overlay network allowing interoperable payments between nodes that may be connected to different payment systems.

For more details see https://interledger.org

### Payment Pointers

Payment Pointers are a convenient and concise way to express a URL to a secure payment initiation endpoint on the Web. 

Payment Pointers resolve to an HTTPS endpoint using simple conversion rules allowing systems that offer payment accounts to users to give them a simple and easy to remember identifier for the account that is **safe to share** with others and is immediately identifiable as a payment account identifier.

An example of a Payment Pointer is: `$alice.wallet.example` or `$wallet.example/alice`
These resolve to `https://alice.wallet.example/.well-known/pay` and `https://wallet.example/alice` respectively.

For more details see https://paymentpointers.org

## Getting Started

### Setup a receiving account

To use Web Monetization a website owner must have a financial account at a service provider capable of receiving payments via the Interledger protocol. 

Such a service (a digital wallet, bank, or similar) would provide the website owner with a _**Payment Pointer**_ that can be used to send payments to that account. 

> **Example:** Alice owns the website at _https://rocknrollblog.example_ and opens an account at _Secure Wallet Ltd._. Secure Wallet tells Alice that the Payment Pointer for her account is `$secure-wallet.example/~alice`.

### Add &lt;meta> tag to website header

The website puts a `<meta>` tag in the header of the HTML documents it serves with the `name` attribute equal to `monetization` and the `value` attribute equal to the Payment Pointer where the website will accept payments.

> **Example:** Alice puts the tag `<meta name="monetization" value="$secure-wallet.example/~alice">` into the `<head>` section of _https://rocknrollblog.example_.

### Handle payments

When a user visits the page with a supported browser the website will find a `document.monetization` object in the DOM. This will have a `state` property that the website can check to determine if the user's provider has started sending payments.

The `document.monetization` object will emit events when monetization starts and then subsequently each time a payment is sent successfully by the provider. The start event will contain a unique identifier for the payment stream that the website can use to correlate the payments at its receiver with the user's browser session.

> **Example:** Alice adds some client-side code to her website that listens for the relevant monetization events and only shows advertising if she isn't receiving payments.

```html
<head>
  <meta name="monetization" value="$secure-wallet.example/~alice">
</head>
<script>
  if(document.monetization) {
    let requestId
    document.monetization.addEventListener('monetizationstart', event => {

      // User has an open a payment stream

      // Connect to backend to validate the session using the request id
      const { paymentPointer, requestId } = event.detail
      if(!isValidSession(paymentPointer, requestId)) {
        console.error('Invalid requestId for monetization')
        showAdvertising()
      }
    })

    document.monetization.addEventListener('monetizationprogress', event => {

      // A payment has been received 

      // Connect to backend to validate the payment
      const { amount, assetCode, assetScale } = event.detail
      if(isValidPayment(paymentPointer, requestId, amount, assetCode, assetScale)) {
        // Hide ads for a period based on amount received
        suspendAdvertising(amount, assetCode, assetScale)
      }
    })
    // Wait 30 seconds and then show ads
    setTimeout(showAdvertising, 30000)
  } else {
    showAdvertising()
  }
</script>
```

### Browser Behaviour


A browser supporting Web Monetization exposes a DOM object `document.monetization` that implements [EventTarget](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) and has a readonly `state` property. Initially the browser sets `document.monetization.state` to `pending`.

 1. If the browser finds a valid Payment Pointer in a Web Monetization `<meta>` tag it generates a fresh UUID (version 4) and uses this as the **Request ID** from this point forward. **This Request ID MUST be unique per page load**, not per browser, session nor site.
     - The `<meta>` tags MUST NOT be inserted dynamically using client-side Javascript.
     - The `<meta>` tags MUST be in the `<head>` of the document.
     - If the Web Monetization `<meta>` tags are malformed, the browser will stop here. The browser SHOULD report a warning via the console.
     - If the Web Monetization `<meta>` tags are well-formed, the browser should extract the Payment Pointer.

 2. The browser invokes the user's Web Monetization Provider passing it the `requestId` and `paymentPointer`.

 3. The provider resolves the Payment Pointer and begins to make payments to the website.

 4. Once the provider has successfully completed the first payment with a non-zero amount, the provider MUST notify the browser, and the browser sets `document.monetization.state` to `started` and then dispatches the `monetizationstart` event on `document.monetization`. The event's type is `monetizationstart`. The event has a `detail` field with an object containing the Payment Pointer and the Request ID ([specified below](#monetizationstart)).

 5. Every time the provider processes a payment (including the first payment) it notifies the browser which dispatches a `monetizationprogress` event from `document.monetization`. The event has a `detail` field with an object containing the amount and currency of the payment.

 6. Payment continues until the user closes/leaves the page. The provider MAY decide to stop/start payment at any time, e.g. if the user is idle or backgrounds the page.


------ TEMPLATE -----

## Key scenarios

Next, discuss the key scenarios which move beyond the most canonical example, showing how they are addressed using example code:

### Scenario 1

Outline the scenario, then provide:

<sample code that demonstrates the feature>

### Scenario 2

Outline the scenario, then provide:

<sample code that demonstrates the feature>


## Detailed design discussion

### Tricky design choice #1

Talk through the tradeoffs in coming to the specific design point you want to make, hopefully:

<illustrated with example code>


### Tricky design choice N





## Considered alternatives

One of the most important things you can do in your design process is to catalog the set of roads not taken. As you iterate on your design, you may find that major choices in your approach or API style will be revisited and enumerating the full space of alternatives can help you apply one (or more) of them later, may serve as a “graveyard” for u-turns in your design, and can give reviewers and potential users confidence that you’ve got your ducks in a row.


## References & acknowledgements

Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.




---------- NOTES ---------

  - How does provider know the page has lost focus?
  - How does the website specify a minimum amount required to access the services?

  


### Why now?

This has been tried before. There is a graveyard of efforts to introduce micropayments to the Web that have all failed. For the most part these have failed for one of the following reasons:

 - User friction: The cognitive load on users agreeing to pay for services is too high. Research suggests that when users can separate the payment from the service (by pre-paying, paying afterwards in aggregate or paying for a subscription service that provided a bundle of content or services) they are far more likely to consume content and services. Web Monetization allows providers to design their business models differently and compete for users based on this and other differentiating factors.

 - 

