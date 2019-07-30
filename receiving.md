# Receiving Payments

When a user has a Web Monetization (WM) provider registered/installed in their browser and they visit a monetized website the browser activates the user's WM provider passing it the receiving address for payments to the website.

The user's provider then begins sending payments to the website's receiving address.

This page describes the functions of the Web Monetization Receiver and how existing payment services might become a WM receiver.

## Receiving Address

The receiving address that a website provides is either a [Payment Pointer](https://paymentpointers.org) or a URL from which the user's WM provider fetches connection details to [open a payment stream](#open-a-stream) with the receiver.

Since the address is a URL (or a Payment Pointer that resolves to a URL) it is possible for the host of the address to redirect the provider to an alternate address using standard HTTP redirects.

## Receiving Payments

Any entity that is able to accept payments on behalf of websites can fulfil the role of being a Web Monetization receiver.

The protocol for establishing a micropayments stream and sending payments is the Interledger protocol. Interledger allows for payments to be sent at very high volumes and in very small denominations.

Payments are sent as Interledger packets which are optimised for this use case.

To accept a stream of micropayments on behalf of its users a WM receiver MUST support 3 protocols from the Interledger stack:

#### Simple Payment Setup Protocol (HTTPS)

The receiver must host a secure HTTP endpoint that can generate unique connection credentials for each WM provider that requests them at this URL.

##### Example: Respond to an SPSP request
 - *Alice* is a customer of *Secure Wallet* who are able to act as a WM Receiver.
 - She is allocated the SPSP endpoint URL `https://securewallet.example/~alice`
 - This URL can also be expressed as a Payment Pointer: `$securewallet.example/~alice`
 - When a GET request is made to `https://securewallet.example/~alice` with the `accept` header of `application/spsp4+json` then the server responds with a JSON response carrying the Interledger address and a shared secret to use to [open a payment stream](#open-a-stream) with *Secure Wallet* and send Alice money.

 More details on SPSP can be found in [IL-RFC 0009 - Simple Payment Setup Protocol](https://interledger.org/rfcs/0009-simple-payment-setup-protocol/).

#### Interledger Protocol

The next service that the receiver MUST setup is a link into the Interledger network. More specifically, the receiver must have an Interledger connection (direct or indirect) to any WM providers that wish to pay the WM receiver.

Initially we expect the number of providers and receivers to be small and they will likely connect to each other directly but in time there will be a need for intermediary aggregators that are licensed to accept payments from providers and aggregate and pay these out to receivers.

#### STREAM Protocol

The STREAM protocol is used to establish packet switched connections between entities using the Interledger protocol. It is loosely based on QUIC and provides similar features over ILP that QUIC does over IP.

For the purposes of being a WM receiver an entity only needs to run a STREAM server, accept connections and handle the incoming payments sent over the connection.

The ILP Address and shared secret acquired by the WM provider through SPSP ([see above](#-Simple-Payment-Setup-Protocol-HTTPS)) is used to establish a STREAM connection with the WM receiver, following which a stream of payments can be made to the receiver until finally the stream is closed by the client.

Each set of connection parameters (ILP Address and shared secret) is unique and allows the receiver to correlate the original SPSP request with the incoming connection. This makes it easier for receivers to associate incoming payments with different user accounts and Web Monetization sessions.
