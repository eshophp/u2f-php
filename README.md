# U2F

A PHP implementation of the FIDO U2F authentication standard

[![Build Status](https://travis-ci.org/Firehed/u2f-php.svg?branch=master)](https://travis-ci.org/Firehed/u2f-php)
[![Coverage Status](https://coveralls.io/repos/github/Firehed/u2f-php/badge.svg)](https://coveralls.io/github/Firehed/u2f-php)

## Introduction

U2F, or Universal Second Factor, is a new authentication protocol designed "to augment the security of their existing password infrastructure by adding a strong second factor to user login"[1](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-u2f-overview.html#background). It allows websites to replace the need for a companion app (such as Google Authenticator) with a single hardware token that will work across any website supporting the U2F protocol.

This library is designed to allow easy integration of the U2F protocol to an existing user authentication scheme.
It handles the parsing and validating all of the raw message formats, and translates them into standard PHP objects.

Note that use of the word "key" throughout this document should be interpreted to mean "FIDO U2F Token".
These are often USB "keys" but can also be NFC or Bluetooth devices.

There are two main operations that you will need to understand for a successful integration: registration and authentication.
Registration is the act of associating a key that the end-user is physically in posession of with their existing account; authentication is where that key is used to cryptographically sign a message from your application to verify posession of said key.

Additional resources:

* [FIDO U2F Overview](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-u2f-overview.html)
* [FIDO U2F Javascript API](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-u2f-javascript-api.html)

## Installation

`composer require firehed/u2f`

Note: you **must not** be using the deprecated `mbstring.func_overload` functionality, which can completely break working on binary data.
The library will immediately throw an exception if you have it enabled.

## Usage

Usage will be described in three parts: setup, registration, and authentication.
The code in setup should be used before both registration and authentication.

The API is designed to "fail loudly"; that is, failures will throw an exception, ensuring that return values are always the result of a successful operation.
This reduces the need for complex error checking and handling during use, since the whole thing can be simply wrapped in a `try/catch` block and assume that everything went well if no exceptions are caught.

This guide covers the modern Web Authentication ("WebAuthn") usage and data formats.
For the legacy u2f.js usage, see [usage-u2f.md](usage-u2f.md).

### Setup

All operations are performed by the U2F Server class, so it needs to be instanciated and configured:

```php
use Firehed\U2F\Server;
$server = new Server();
$server->setTrustedCAs(glob('path/to/certs/*.pem'))
       ->setAppId('https://u2f.example.com');
```

The trusted CAs are whitelisted vendors, and must be an array of absolute paths to PEM-formatted CA certs (as strings).
Some provider certificates are provided in the `CACerts/` directory in the repository root; in a deployed project, these should be available via `$PROJECT_ROOT/vendor/firehed/u2f/CACerts/*.pem`.

You may also choose to disable CA verification, by calling `->disableCAVerification()` instead of `setTrustedCAs()`.
This removes trust in the hardware vendors, but ensures that as new vendors issue tokens, they will be forward-compatible with your website.

The URI provided to `setAppId()` must be the HTTPS domain component of your website.
See [FIDO U2F AppID and Facet Specification](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-appid-and-facets.html#appid-example-1) for additional information.

### Registration

Registering a token to a user's account is a two-step process: generating a challenge, and verifying the response to that challenge.

#### Generating the challenge

Start by generating a challenge.
You will need to store this temporarily (e.g. in a session), then send it to the user:

```php
$request = $server->generateRegisterRequest();
$_SESSION['registration_request'] = $request;

header('Content-type: application/json');
echo json_encode($request->getChallenge());
```

Client-side, create a [`PublicKeyCredentialCreationOptions`](https://www.w3.org/TR/webauthn/#dictdef-publickeycredentialcreationoptions) data structure, and provide it to the WebAuthn API:

```js
const userId = "some value from your application"
const challenge = "challenge string from above"
const options = {
  rp: {
    name: "Example Site",
  },
  user: {
    id: Uint8Array.from(userId, c => c.charCodeAt(0)),
    name: "user@example.com",
    displayName: "User Name",
  },
  challenge: Uint8Array.from(challenge, c => c.charCodeAt(0)),
  pubKeyCredParams: [{alg: -7, type: "public-key"}],
  timeout: 60000, // 60 seconds
  authenticatorSelection: {
    authenticatorAttachment: "cross-platform",
    userVerification: "preferred",
  },
  attestation: "direct"
}

// If the user completes registration, this value will hold the data to POST to your application
const credential = await navigator.credentials.create({
  publicKey: options
})
```

Format the user's `credential` and POST it to your application:

```js
const dataToSend = {
  rawId: new Uint8Array(credential.rawId),
  type: credential.type,
  response: {
    attestationObject: new Uint8Array(credential.response.attestationObject),
    clientDataJSON: new Uint8Array(credential.response.clientDataJSON),
  },
}

// Pseudocode:
// POST will send JSON.stringify(dataToSend) with an application/json Content-type header
const response = POST('/verifyRegisterChallenge.php', dataToSend)
```

#### Parse and verify the response

```php
// You should validate that the inbound request has an 'application/json' Content-type header
$rawPostBody = trim(file_get_contents('php://input'));
$data = json_decode($rawPostBody, true);
$response = \Firehed\U2F\WebAuthn\RegistrationResponse::fromDecodedJson($data);

$server->setRegisterRequest($previously_generated_request);
$registration = $server->register($response)
```

##### Persist the `$registration`

Registrations SHOULD be persisted as a one-to-many relationship with the user, since a user may own multiple keys and may want to associate all of them with their account (e.g. a backup key is kept on a spouse's keychain).
It is RECOMMENDED to use `(user_id, key_handle)` as a unique composite identifier.

```sql
-- A schema with roughly this format is ideal
CREATE TABLE token_registrations (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    counter INTEGER,
    key_handle TEXT,
    public_key TEXT,
    attestation_certificate TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id),
    UNIQUE(user_id, key_handle)
)
```
```php
$stmt = $pdo->prepare('
    INSERT INTO token_registrations (
        user_id,
        counter,
        key_handle,
        public_key,
        attestation_certificate
    ) VALUES (
        :user_id,
        :counter,
        :key_handle,
        :public_key,
        :attestation_certificate
    )
');
// Note: you may want to base64- or hex-encode the binary values below.
// Doing so is entirely optional.
$stmt->execute([
    ':user_id' => $_SESSION['user_id'],
    ':counter' => $registration->getCounter(),
    ':key_handle' => $registration->getKeyHandleBinary(),
    ':public_key' => $registration->getPublicKey()->getBinary(),
    ':attestation_certificate' => $registration->getAttestationCertificate()->getBinary(),
]);
```

### Authentication

Authentication is a similar process as registration: generate challenges to sign for each of the user's registrations, and validate the response when received.

#### Generating the challenge

Start by generating sign requests.
Like with registration, you will need to store them temporarily for verification.
After doing so, send them to the user:

```php
$registrations = $user->getU2FRegistrations(); // this must be an array of Registration objects

$signRequests = $server->generateSignRequests($registrations);
$_SESSION['sign_requests'] = $signRequests;

// WebAuthn expects a single challenge for all key handles, and the Server generates the requests accordingly.
header('Content-type: application/json');
echo json_encode([
    'challenge' => $signRequests[0]->getChallenge(),
    'key_handles' => array_map(function (SignRequest $sr) {
        return $sr->getKeyHandleWeb();
    }, $signRequests),
]);
```

Client-side, turn the challenge and key handles into a [`PublicKeyCredentialRequestOptions`](https://www.w3.org/TR/webauthn/#assertion-options) data structure, and provide it to the WebAuthn API:

```js
// This is a basic decoder for the above `getKeyHandleWeb()` format
const fromBase64Web = s => atob(s.replace(/\-/g,'+').replace(/_/g,'/'))

const challenge = postedData.challenge
const keyHandles = postedData.key_handles
const options = {
  challenge: Uint8Array.from(challenge, c => c.charCodeAt(0)),
  allowCredentials: keyHandles.map(kh => ({
    id: Uint8Array.from(fromBase64Web(kh), c => c.charCodeAt(0)),
    type: 'public-key',
    transports: ['usb', 'ble', 'nfc'],
  })),
  timeout: 60000,
}
// If the user authenticates, this value will hold the data to POST to your application
const assertion = await navigator.credentials.get({
  publicKey: options
});
```

Format the user's `assertion` and POST it to your application:
```js
const dataToSend = {
  rawId: new Uint8Array(assertion.rawId),
  type: assertion.type,
  response: {
    authenticatorData: new Uint8Array(assertion.response.authenticatorData),
    clientDataJSON: new Uint8Array(assertion.response.clientDataJSON),
    signature: new Uint8Array(assertion.response.signature),
  },
}

// Pseudocode, same as above
const response = await POST('/verifyLoginChallenge.php', dataToSend)
```
#### Parse and verify the response

Parse the POSTed data into a `LoginResponseInterface`:
```php
// You should validate that the inbound request has an 'application/json' Content-type header
$rawPostBody = trim(file_get_contents('php://input'));
$data = json_decode($rawPostBody, true);
$response = \Firehed\U2F\WebAuthn\LoginResponse::fromDecodedJson($data);

$registrations = $user->getU2FRegistrations(); // Registration[]
$server->setRegistrations($registrations)
       ->setSignRequests($_SESSION['sign_requests']);
$registration = $server->authenticate($response);
```

#### Persist the updated `$registration`
If no exception is thrown, `$registration` will be a Registration object with an updated `counter`; you MUST persist this updated counter to wherever the registrations are stored.
Failure to do so **is insecure** and exposes your application to token cloning attacks.

```php
$stmt = $pdo->prepare('
    UPDATE token_registrations
    SET counter = :counter
    WHERE user_id = :user_id
        AND key_handle = :key_handle
');
$stmt->execute([
    ':counter' => $registration->getCounter(),
    ':user_id' => $_SESSION['user_id'],
    ':key_handle' => $registration->getKeyHandleBinary(), // if you are storing base64- or hex- encoded above, do so here as well
]);
```

<!-- live demo is not updated for webauthn yet -->
<!--
## Demo

You may try all of this at https://u2f.ericstern.com, and see the corresponding code at https://github.com/Firehed/u2f-php-examples

The example code is only designed to show how the APIs interact with each other, and intentionally leaves out best practices such as use of routers and dependency inversion containers to keep the examples as simple as possible.
See its README for more information.
-->

## Tests

All tests are in the `tests/` directory and can be run with `vendor/bin/phpunit tests/`.

## License

MIT
