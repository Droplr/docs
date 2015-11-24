# Public Operations

## Authentication & Authorization

All API operations must be properly authenticated. In order for you to access the API you must have a public/private key combination to access Droplr's API server.

Authentication works on a per-request basis, which means that every single request must be pre-signed according to its contents.

Droplr API server uses a custom authentication method along with some other mechanisms to ensure a safe non-reversible authentication method which is also immunte to replay attacks.


### Protection Against Replay Attacks

All requests must include a `Date` header, with the time in Unix (POSIX) format -- the **milliseconds** elapsed since midnight January 1st, 1970, UTC.

This value must fall within 15 minutes (ahead or behind) the server's clock. If this condition fails, the requests may be discarded. Furthermore, the server keeps used signatures in a local cache -- during the time window in which they are valid -- in order to avoid replay attacks.

<aside class="notice">
  If the framework you're using doesn't allow you to manually set the <code>Date</code> header, you can use the custom <code>x-droplr-date</code> header. When set, this header will take precedence over the <code>Date</code> header.
</aside>


### Application Privileges

Droplr API server provides many operations; the ability to execute them depends on the privileges configured for the public/private API key pair assigned to an application.

As an example, a typical third party application will be able to list drops, perform uploads, delete drops and alter user information but it won't be able to create new accounts.


### Authentication Formula

```shell
# Example Authorization header

Authorization: droplr YXBwXzBfcHVibGlja2V5OnVzZXJfMUBkcm9wbHIuY29t:3+MqSMFYYwh6grneUezBtRkunmE=
```

Starting from the end, an example of how an `Authorization` header should look is:

`Authorization: droplr YXBwXzBfcHVibGlja2V5OnVzZXJfMUBkcm9wbHIuY29t:3+MqSMFYYwh6grneUezBtRkunmE=`

This can be decomposed as:

`Authorization: droplr BASE64(ApplicationPublicKey:UserEmail):signature`

Where `ApplicationPublicKey` and `UserEmail` are your application's assigned public key and the user's email.
The formula to compute the `signature` parameter is:

`HMAC_SHA1(ApplicationPrivateKey:UserPasswordSHA1, stringToSign)`

`ApplicationPrivateKey` and `UserPasswordSHA1` are your application's assigned secret key and the user's (hashed) password.
The hashing function for the password is SHA-1.

<aside class="notice">
  We <strong>strongly recommend</strong> you to always store the hash of the password as your users enter their credentials, discarding the clear password as soon as possible.
</aside>

```shell
# Example stringToSign parameter

POST /note.json HTTP/1.1\n
text/plain\n
1299648278321


# Example stringToSign parameter with no contentType

GET /files/code HTTP/1.1\n
\n
1299648278321


# Putting it all together

stringToSign = Request-Line\n
               Content-Type\n
               Date

signature = HMAC_SHA1(ApplicationPrivateKey:MD5(UserPassword), stringToSign)

Authorization: droplr BASE64(ApplicationPublicKey:UserEmail):signature
```

The `stringToSign` parameter is a concatenation of some of the contents of the request.

The parameter `requestLine` is simply the concatenation of the method, URI and HTTP version (with spaces):

`GET /files/code HTTP/1.1`

`contentType` and `date` parameters are the `Content-Type` and `Date` headers included in the request. While `contentType` may be an empty string (for requests that bear no body), `date` is mandatory and must have a value.

<aside class="warning">
  Even when <code>contentType</code> is empty, its trailing line break <strong>MUST</strong> be included. 
</aside>


### Examples

This sub-section provides a couple of examples of requests and their respective generated signature.

The following credentials will be assumed throughout the examples:

* **Public application key:** `family_app`
* **Private application key:** `quahog`
* **User email:** `quagmire@droplr.com`
* **User password:** `giggity` (will be hashed to `1869bfcf575c810780534a7f5e4f6c225b4ca3bd`)

<aside class="warning">
  The pair of characters <code>\n</code> represents the newline character. Do not escape the newline character when creating the signature.
</aside>


#### Signature Example 1: Reading Account Details (JSON)

Request:

* GET /account.json HTTP/1.1
* Date: 1335230330353
* Content-Length: 0

Signature generation stages:

1. Access key: `ZmFtaWx5X2FwcDpxdWFnbWlyZUBkcm9wbHIuY29t`
2. Access secret: `quahog:1869bfcf575c810780534a7f5e4f6c225b4ca3bd`
3. String to sign: `GET /account.json HTTP/1.1\n\n1335230330353`
4. Signature: `1cGqXOeNPRM5PPpDl1Ca/DdWesY=`

Expected value of `Authorization` header:
  
`droplr ZmFtaWx5X2FwcDpxdWFnbWlyZUBkcm9wbHIuY29t:1cGqXOeNPRM5PPpDl1Ca/DdWesY=`


#### Signature Example 2: Creating a New Note (JSON)

Request:

* POST /notes.json HTTP/1.1
* Date: 1335228853999
* Content-Type: text/plain
* Content-Length: 52
* (... POST body ...) 

Signature generation stages:

1. Access key: `ZmFtaWx5X2FwcDpxdWFnbWlyZUBkcm9wbHIuY29t`
2. Access secret: `quahog:1869bfcf575c810780534a7f5e4f6c225b4ca3bd`
3. String to sign: `POST /notes.json HTTP/1.1\ntext/plain\n1335229121561`
4. Signature: `zwVsqm6VhEGzFhqBQM+zzvh/PJ8=`

Expected value of `Authorization` header:

`droplr ZmFtaWx5X2FwcDpxdWFnbWlyZUBkcm9wbHIuY29t:zwVsqm6VhEGzFhqBQM+zzvh/PJ8=`