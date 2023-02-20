---
title: "GNAP Secure Payment Confirmation Extension"
category: info

docname: draft-ozdemir-gnap-spc-extension-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Grant Negotiation and Authorization Protocol"
keyword:
 - gnap
 - secure payment confirmation
 - extension
venue:
  group: "Grant Negotiation and Authorization Protocol"
  type: "Working Group"
  mail: "txauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/txauth/"
  github: "fynbos-dev/gnap-spc-extension"
  latest: "https://fynbos-dev.github.io/gnap-spc-extension/draft-ozdemir-gnap-spc-extension.html"

author:
 -
    fullname: Omer Talha Ozdemir
    organization: Fynbos
    email: omer@fynbos.dev
 -
    fullname: Adrian Hope-Bailie
    organization: Fynbos
    email: adrian@fynbos.dev

normative:
  I-D.ietf-gnap-core-protocol-12: GNAP
  W3C.secure-payment-confirmation: SPC
  W3C.webauthn-3: WebAuthn

informative:


--- abstract

GNAP Secure Payment Confirmation (SPC) Extension is a GNAP extension that defines a method for authentication of the end user during a payment transaction. This extension helps leverage hardware and software authenticators such as biometric scanners while authenticating the end user.

--- middle

# Introduction

GNAP Secure Payment Confirmation {{-SPC}} Extension is an extension developed on top of Grant Negotiation and Agreement Protocol {{-GNAP}}. It defines a method for authentication of the end user during a payment transaction using Secure Payment Confirmation. This extension helps leverage authenticators such as fingerprint scanners, facial recognition systems, etc. while authenticating the end user in a GNAP exchange.

# Checking Feature Support

This extension only works if the end user's user agent supports Payment Request API and SPC. To detect whether SPC is supported on the browser, client instance can send a fake call to `canMakePayment()`.

The following code provides a feature detect function for the Payment Request API and the SPC on a merchant's website.

~~~ javascript
const isSecurePaymentConfirmationSupported = async () => {
  if (!'PaymentRequest' in window) {
    return [false, 'Payment Request API is not supported'];
  }

  try {
    // The data below is the minimum required to create the request and
    // check if a payment can be made.
    const supportedInstruments = [
      {
        supportedMethods: "secure-payment-confirmation",
        data: {
          // RP's hostname as its ID
          rpId: 'rp.example',
          // A dummy credential ID
          credentialIds: [new Uint8Array(1)],
          // A dummy challenge
          challenge: new Uint8Array(1),
          instrument: {
            // Non-empty display name string
            displayName: ' ',
            // Transparent-black pixel.
            icon: 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFhAJ/wlseKgAAAABJRU5ErkJggg==',
          },
          // A dummy merchant origin
          payeeOrigin: 'https://non-existent.example',
        }
      }
    ];

    const details = {
      // Dummy shopping details
      total: {label: 'Total', amount: {currency: 'USD', value: '0'}},
    };

    const request = new PaymentRequest(supportedInstruments, details);
    const canMakePayment = await request.canMakePayment();
    return [canMakePayment, canMakePayment ? '' : 'SPC is not available'];
  } catch (error) {
    console.error(error);
    return [false, error.message];
  }
};
~~~

# Requesting Credentials

A client that wishes to request SPC credentials from authorization server MUST request a grant from authorization server with SPC as its preferred interaction method. To request a grant, the client sends an HTTP POST with a JSON document to the grant endpoint of the AS. The JSON object is a GNAP grant request that uses SPC as its interaction start method. SPC extension adds `spc` value to the `start` field in the `interact` object. The client **MUST** identify the end user using the `user` property in the grant request object to allow AS find the registered credentials for the user.

A non-normative example of a grant request that uses SPC as its interaction start method is below.

~~~ json
"access_token": {
  "access": [{
    "type": "outgoing-payment",
    "actions": [
      "create",
      "read",
      "list"
    ],
  }]
},
"client": {
    "key": {
      "proof": "httpsig",
      "jwk": {
        "alg": "EdDSA",
        "kty": "OKP",
        "use": "sig",
        "crv": "Ed25519",
        "kid": "xyz-1",
        "x": "jfiusdhvherui..."
        }
    }
  },
"interact": {
  "start": [
    "spc"
  ]
},
"user": {
  "sub_ids": [{
    "subject_type": "email",
    "email": "user@example.com"
  }]
}
~~~

# Serving Credentials

In response to a client instance’s grant request, if the AS determines that it has a registered SPC credential of the end user, the AS responds with JSON object as the HTTP body. AS uses the `spc` field in the `interact` object to provide necessary information to initiate a SPC authentication. This field contains a JSON object with the following properties.

`credential_ids` (array of strings):
: Each credential MUST be base64url encoded. **REQUIRED**

`challenge` (string):
: A random challenge that the relying party generates on the server side to prevent replay attacks. The challenge MUST be base64url encoded. **REQUIRED**.

A non-normative example of a grant request continue response that uses SPC as its interaction method is below.

~~~ json
{
  "interact": {
    "spc": {
      "credential_ids": ["MTIzMjMxMzIyMz..."],
      "challenge": "dGhpcyBpcyBh..."
    }
  },
  "continue": {
    "access_token": {
      "value": "80UPRY5NM33OMUKMKSKU"
    },
    "uri": "http://wallet.com/adrian/continue/5e69f364-b14d-4fdf-8b6b-3b6ffb52c339"
  }
}
~~~

# Authenticating User
In response to the AS's grant request response, the client should initiate the authentication ceremony performing the steps as specified in {{-SPC}}. The client instance should decode the `challenge` and each credential from `credential_ids` using base64url and convert them to buffer.

# Completing Interaction
The client **MUST** encode each property of the Web Authentication Assetion object {{-WebAuthn}} returned from successful authentication ceremony using base64url.

`clientDataJSON` (string):
: `clientDataJSON` property from Web Authentication Assertion object. This **MUST** encoded using base64url. **REQUIRED**.

`authenticatorData` (string):
: `authenticatorData` property from Web Authentication Assertion object. This **MUST** encoded using base64url. **REQUIRED**.

`signature` (string):
: `signature` property from Web Authentication Assertion object. This **MUST** encoded using base64url. **REQUIRED**.

`userHandle` (string):
: `userHandle` property from Web Authentication Assertion object. This **MUST** encoded using base64url. **OPTIONAL**.

A non-normative example of a interaction completion response is below.

~~~ json
"public_key_cred": {
  "clientDataJSON": "ZXhhbXBsZSBjbGllbnRkYXR...",
  "authenticatorData": "YXV0aGVudGljYXRvckRhdGEg...",
  "signature": "c2lnbmF0dXJlIGV4YW...",
  "userHandle": "dXNlckhhbmRsZSBleG..."
}
~~~

# Verifying Authentication Assertion
In order to complete authentication ceremony and authenticate the end user, the AS **MUST** perform the steps specified in Section 8.1 of Secure Payment Confirmation. The AS should decode each property of public key credential in the response using base64url before performing the verification. When verification is completed successfully, the AS can grant access and return a proper grant response.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.