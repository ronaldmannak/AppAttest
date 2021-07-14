# AppAttest

The App Attest service, which Apple introduced in iOS 14, provides a secure way of verifying that connections to your server come from legitimate instances of your app. Generating assertions and attestations in your app is [fairly straightforward](https://developer.apple.com/documentation/devicecheck/establishing_your_app_s_integrity), but verifying them on the server is [a little more complicated](https://developer.apple.com/documentation/devicecheck/validating_apps_that_connect_to_your_server). This Swift package implements the server-side validation logic for you.

Note that this is still a young project and the API may change a little. At the moment, the library is able to validate assertions and attestations, but not receipts (which is an optional step, anyway).


## Installation

AppAttest is distributed with the [Swift Package Manager](https://swift.org/package-manager/). To install it, just add the following dependency within your `Package.swift` manifest:

``` Swift
let package = Package(
    ...
    dependencies: [
        .package(url: "https://github.com/iansampson/AppAttest", .branch("main"))
    ],
    ...
)
```


## Usage

This example shows how to validate assertions and attestations on your server. The details of how to send data back and forth to your app (via HTTP requests and responses, for example) and handling state (such as storing challenges and assertions between server calls) are left as an exercise to the reader. For simplicity, code snippets are written at file scope, though you almost certainly want to wrap them inside functions.


### Generate a challenge

First, your app sends a request to the server (for example, trying to access some content) and the server responds with a challenge. The challenge should be random, unguessable, and unique (i.e. used only once per attestation). You could use[swift-crypto](https://github.com/apple/swift-crypto), for example, to generate a nonce:

``` Swift
import Crypto

let challenge = Data(AES.GCM.Nonce())
```

Send this challenge to your app as part of the HTTP response and store it on your server (either in memory or in a database). You’ll need to retrieve the response in the next step. (A simple way to do this is to generate a UUID for the challenge. Include the ID as part of the HTTP response and store it in a dictionary.)

``` Swift
var challenges: [UUID: Data] = [:]

let challengeID = UUID()
challenges[challengeID] = challengeID
```


### Verify an attestation

When your app receives this challenge, use the DCAppAttestService to generate an attestation.

The app will also need to generate a key if it hasn’t already done so.

``` Swift
import CryptoKit

service.generateKey { keyId, error in
    guard error == nil else {
      // Handle the error
    }
    // Store keyId for subsequent operations
}

let service = DCAppAttestService.shared

let challenge = ... // Challenge from your server
let hash = Data(SHA256.hash(data: challenge))

service.attestKey(keyId, clientDataHash: hash) { attestation, error in
    guard error == nil else {
      // Handle error
    }
    // Send the attestation to your server for verification
}
```

Your app sends the attestation to your server (along with the key ID and the challenge ID, if you generated one in the first step). You’ll also need your 10-digit team ID (which you can find in App Store Connect) and your app’s bundle ID. Your server then calls the static AppAttest.verifyAttestation method to, well, verify the attestation.

``` Swift
import AppAttest

// Retrieve these values from the HTTP request
// that your app sends to the server
let attestation: Data = ...
let keyID: Data = ...
let challengeID: UUID = ...

// Retrieve the challenge you generated in the previous step
let challenge = challenges[challengeID]

// Construct the attestation response and app ID,
// which are simple structs
let response = AttestationResponse(attestation: attestation, keyID: keyID)
let appID = AppID(teamID: "83Z139DVZ2", bundleID: "com.example.myapp")

// Verify the attestation
do {
    let result = try AppAttest.verifyAttestation(challenge: challenge, response, appID: appID)
} catch {
    // Handle the error
}
```

If attestation succeeds, the result contains the public key generated by Apple (the same one your app stores locally) and the attestation receipt. Store these on your server for later use (e.g. in a dictionary keyed by the key ID sent by the client). Let your app know that attestation was successful.


### Verify an assertion

Verifying assertions follows a similar process. Your app sends a request to the server and the server responds with a (new) challenge. Your app uses this challenge to generate an assertion.

``` Swift
import CryptoKit

let challenge = // Challenge from your server
let hash = Data(SHA256.hash(data: clientData))
// For more complex assertions, you can also embed
// the challenge in a JSON payload along with other data

service.generateAssertion(keyId, clientDataHash: clientDataHash) { assertion, error in
    guard error == nil else {
        // Handle the error
    }
    // Send the assertion and request to your server
}
```

Your app sends the assertion, your key ID, and (optionally) the challenge ID to your server, which verifies the assertion.

``` Swift

// Retrieve these values from the HTTP request
// that your app sends to the server
let assertion: Data = ...
let keyID: Data = ...
let challengeID: UUID = ...

// Retrieve the challenge you generated in the previous step
let challenge = challenges[challengeID]

// Retrieve the attestation you stored previously
let attestation = ...

// If this is not the first assertion for this instance
// of this app (i.e. for this unique key ID),
// retrieve the previous AssertionResult. Otherwise,
// use nil for this value.
let previousAssertion = ...

// Construct the assertion response
let response = AppAttest.AssertionResponse(clientResponse: assertion)

do {
    let result = try AppAttest.verifyAssertion(challenge: challenge,
                                               response: response,
                                               previousResult: previousAssertion,
                                               publicKey: attestation.publicKey,
                                               appID: appID
    )
} catch {
  // Handle error
}
```

If assertion succeeds, store the result (e.g. in a dictionary keyed by the key ID sent by the client) and respond to your app (e.g. with some requested content).
