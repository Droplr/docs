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

## Data Formats

You'll need to provide an input for many of the operations against the Droplr API servers, and most operations will also return an output.

Two types of output are supported:

* Custom headers (from here on referred to as HEADERS, which are headers with the prefix `x-droplr-*`)
* JSON

Whenever possible, clients should use the HEADERS output format. It's lighter for both the server and the client because there is no need for encoding/decoding the contents.

While HEADERS format is perfect for conveying data on a single record it is inadequate for large and/or multi-record responses. This means that some actions, given the nature of their responses, are not available with HEADERS format -- as is the case with list operations.

The choice of output format is done by appending a dot (`.`) followed by the desired format at the end of the request URI. For the default, HEADERS format, however, nothing is appended.

Examples:

* `/drops/xkcd.json` (JSON)
* `/drops/xkcd` (HEADERS)

### Input & Output Coherence

> HEADERS Output Format

```shell
# Request (partial):

POST /note HTTP/1.1
...
(request body)

# Response (partial):

HTTP/1.1 200 OK
x-droplr-code: xkcd
x-droplr-uploadsize: 69
x-droplr-availablespace: 1000
x-droplr-usedspace: 169
...
Content-Length: 0
```

> JSON Output Format

```shell
# Request (partial):

POST /note.json HTTP/1.1
...
Content-Type: application/json
Content-Length: 461
...
(request body)

# Response:

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 78

{"totalSpace":1073741824,"usedSpace":1400519,"uploadSize":461,"code":"xkcd", ...}
```

> JSON Output Format with Query Input

```shell
# Request (partial):

GET /drops.json?offset=0&amount=10 HTTP/1.1
...
Content-Length: 0

# Response:

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 78

{"totalSpace":1073741824,"usedSpace":1400519,"uploadSize":461,"code":"xkcd", ...}
```

When executing an operation, keep in mind that the server will give output according to the input. If you access the operation `GET /drops/xkcd`, the server expects you to provide input using the HEADERS format and will provide output with the HEADERS format. Unless explicitly noted, data format permutations cannot occur; for example, you can't execute `GET /drops/xkcd` with input in HEADERS format and receive data in JSON format.

There are, however, HTTP client libraries that will not allow you to send data along in a GET request. In order to overcome this limitation, all JSON operations that support or require input parameters will also support query parameter input:

`GET /drops.json?offset=0&amount=10 HTTP/1.1`

The following examples illustrate the differences between these output formats for the example of a note upload.

<aside class="notice">
  Requests in the HEADERS format do not require the format type to be appended at the end of the URI (as is the case with all other formats).
</aside>


## Operations Reference

Droplr's API consists of several RESTful operations to create, read and delete drops as well as operations to manage user accounts.

This document provides an extensive listing of these operations, along with their input & output parameters, supported [formats](#data-formats), and special notes.


### Environments and Encryption

Before any application is authorized to hit the production servers, it must go through a staging period on the development environments.

Server endpoints:

* **Production**: https://api.droplr.com (port 443)
* **Development**: http://dev.droplr.com (port 8069)

While encrypted HTTP connections are optional on the development environment, they are mandatory for the production environment -- which doesn't even support non-encrypted connections.


### Operation Errors

```shell
# Example error response

HTTP/1.1 401 Unauthorized
...
x-droplr-errorcode: Authentication.UnknownUser
x-droplr-errordetails: No such user
```

The server will use traditional HTTP status codes to convey operation errors or success. However, these are somewhat limited and do not offer a precise cause for the operation failure.

Whenever an operation fails (HTTP status code >= 400), the server will include two headers:

* **Error code:** Contains a unique identifier for the cause of the error. It is usually in the format X.Y, X being the high-level action and Y the error identifier, within X. It can be found on the header `x-droplr-errorcode`.

* **Error details:** Contains a user-friendly english (`en-US`) message that can be displayed to the user of the app. It can be found on the header `x-droplr-errordetails`.

#### Common Errors

There are a great deal of errors that are common to all operations. The following is a comprehensive list of these errors along with the matching HTTP status code, a description of the cause and the message that is sent in the `x-droplr-errordetails` header.

* **Internal.DataAccessError**
    * **HTTP status code:** 503 (Service Unavailable)
    * **Cause:** Temporary failure on Droplr's data storage system.
    * **Error details message:** "Temporary data access failure when performing operation"

* **Internal.TooManyRequest**
    * **HTTP status code:** 503 (Service Unavailable)
    * **Cause:** Server is under extremely heavy load and will discard some requests in order to keep serving a percentage of those.
    * **Error details message:** "Server is under heavy load; please try again later"

* **Maintenance.GoingDown**
    * **HTTP status code:** 503 (Service unavailable)
    * **Cause:** Server is under maintenance and will be shutdown soon.
    * **Error details message:** "Server will be going down for maintenance shortly"

* **Maintenance.ReadOnly**
    * **HTTP status code:** 503 (Service unavailable)
    * **Cause:** Server is under maintenance and only non-idempotent operations (read-only) will be allowed.
    * **Error details message:** "Server is under maintenance; only read operations will be allowed"

* **Authentication.InvalidAuthHeader**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** Auth header is invalid; please refer to the "Auth & auth" guide.
    * **Error details message:** "Authorization header format is not in conformity with specification"

* **Authentication.UnknownScheme**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** Authentication scheme is unknown; please refer to the "Auth & auth" guide.
    * **Error details message:** "Authentication scheme not supported: %SCHEME%"

* **Authentication.InvalidScheme**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** The authentication scheme used was not valid for the requested operation.
    * **Error details message:** "Authentication scheme not supported for this action"

* **Authentication.InvalidSignature**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** The HMAC-SHA1 signature does not match expected value.
    * **Error details message:** "HMAC SHA1 signature is invalid"

* **Authentication.ReplayedSignature**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** The signature generated for the request has recently been used. Make sure the value of the *Date* header is being set according to the "Auth & auth" guide.
    * **Error details message:** "Signature has already been used"

* **Authentication.ClockSkew**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** The client's clock is too far ahead/behind the server's clock.
    * **Error details message:** "Date in request (%REQUEST_DATE%) is too far ahead/behind the server date (%SERVER_DATE%)"

* **Authentication.UnknownApplication**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** Application does not have credentials to perform operations on the server.
    * **Error details message:** "No such application"

* **Authentication.UnknownUser**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** User with given identifier does not exist.
    * **Error details message:** "No such user"

* **Authentication.SignatureMismatch**
    * **HTTP status code:** 401 (Unauthorized)
    * **Cause:** Signature is according to spec but result does not match expected. Typically means wrong password.
    * **Error details message:** "Invalid password"

* **Authorization.NoPrivileges**
    * **HTTP status code:** 403 (Forbidden)
    * **Cause:** Application attempted to execute an operation to which it does not have clearance.
    * **Error details message:** "This application has no permissions to execute this action"

* **Request.InvalidUri**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Malformed uri and/or query parameters on HTTP request line.
    * **Error details message:** "Invalid uri and/or query params"

* **Request.NoAction**
    * **HTTP status code:** 404 (Not found)
    * **Cause:** No operation matching request URI.
    * **Error details message:** "No action at the requested uri"

* **Request.BodyMustBeEmpty**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** The application sent body for a request that doesn't require body. Server will immediately reply with error and close the connection to avoid unnecessary resource usage.
    * **Error details message:** "Request body must be empty"

* **Request.NoContentLength**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Client application sent content without specifying a *Content-Lenght* header.
    * **Error details message:** "This server always requires Content-Length header, even for chunked requests"

* **Request.ContentTooLarge**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Client application specified a value that exceeds the acceptable limits (2GB).
    * **Error details message:** "Content-Length indicates illegal size (over 2GB)"

* **Request.NoContentType**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Request did not bear a *Content-Type* header, which is mandatory when the request includes a body.
    * **Error details message:** "Content-Type header is mandatory"

* **Request.ContentLengthMismatch**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Announced content length (on *Content-Length* header) doesn't match actual content length.
    * **Error details message:** "Mismatch between announced content length and actual readable data"

* **Request.NoDateHeader**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** No *Date* (or *x-droplr-date*) header found on request. This header is always mandatory.
    * **Error details message:** "No Date header found in request"

* **Request.NoAuthorizationHeader**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** No *Authorization* header found on request. This header is always mandatory.
    * **Error details message:** "No Authorization header found in request"

* **Request.InvalidJson**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** JSON content in request body could not be correctly decoded.
    * **Error details message:** "Failed to parse JSON request body"

* **Request.UnsupportedDataFormat**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Client application tried to perform an operation using an invalid data format (e.g. ".xml").
    * **Error details message:** "Unsupported request data format: %DATA_FORMAT%"

* **Request.NoJson**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Server was expecting JSON content in request body but no content (or non-JSON content) was found.
    * **Error details message:** "A JSON body is required, along with content Content-Type header set"

* **Request.BadContentType**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Value of the header *Content-Type* is not a valid mime type.
    * **Error details message:** "Unable to parse Content-Type header value: %CONTENT_TYPE%"

* **Request.ChunksNotAcceptable**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Chunked transfer not acceptable for the requested operation.
    * **Error details message:** "Chunked requests not accepted for this action"

* **Request.PipelineAbuse**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** Client application exceeded maximum allowed pipelined requests for a single connection.
    * **Error details message:** "Too many pipelined requests"

* **Request.BlacklistedUserAgent**
    * **HTTP status code:** 403 (Forbidden)
    * **Cause:** The client user agent was blacklisted. Blacklisting user agents is an effective way to block a specific version of an application without deleting its access credentials. Your application may be blacklisted if erratic behavior is detected. Since we assume this is a temporary issue, we convey an error message that warns users to update their application to the latest version.
    * **Error details message:** "Please update to the newest version"

The following errors apply only to drop creation operations (shorten link, create note and upload file).

* **CreateDrop.TooManyUploads**
    * **HTTP status code:** 503 (Service unavailable)
    * **Cause:** Too many concurrent uploads and the server needs to reject some in order to maintain an acceptable bandwidth for all uploads.
    * **Error details message:** "Cannot process your request at this time, please try again later"

* **CreateDrop.MaxSizeExceeded**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** The maximum upload size was exceeded. Drops have a size limit based on type and the user's account
    * **Error details message:** "Max upload size limit exceeded: %SIZE%"

* **CreateDrop.NoSpace**
    * **HTTP status code:** 507 (Insufficient Storage)
    * **Cause:** The user has hit the limits of his account and cannot upload files until he clears out some space.
    * **Error details message:** "Used %USED% of available %AVAILABLE%"

* **CreateDrop.ContentTypeMustMatch**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** The value in *Content-Type* header does not match any of the values in the content type white list.
    * **Error details message:** "Content-Type header is mandatory and must match %LIST%"

* **CreateDrop.InvalidPrivacy**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** The value in *privacy* query param or *x-droplr-privacy* custom header is not a valid privacy value.
    * **Error details message:** "Invalid privacy value"

* **CreateDrop.InvalidPrivacy**
    * **HTTP status code:** 400 (Bad request)
    * **Cause:** The value in *password* query param or *x-droplr-password* custom header is not a valid password value; it either has invalid length (too short or too long) or contains invalid characters.
    * **Error details message:** "Invalid password value"


#### Error Internationalization

```shell
[English dictionary]
(K: Authentication.UnknownUser, V: "There is no such user")

[Portuguese dictionary]
(K: Authentication.UnknownUser, V: "O utilizador não existe")
```

The `error code` property is useful to internationalize your application. By creating multiple dictionaries (also commonly referred to as 'map') where the keys are all the values for the `error code` property and the values are translations, you can show your users a detailed localized message for every error that happens when interacting with the Droplr server.


### Setting a Meaningful User-Agent Header Value

Each application will be uniquely defined by its public key, which will be present in the signature it sends with each request (the value of the `Authorization` header). This identifier will always be the same even though the application version will likely change over time.

In order for Droplr's API servers to be able to distinguish between different versions of the same application, the `User-Agent` header should be set with a meaningful value.

A common and useful pattern is:

`User-Agent: <Network library name>/<major.minor(.build)> <App name>/<major.minor(.build)>`

As an example, Droplr's Mac app using `DroplrKit` would have the following user agent:

`User-Agent: DroplrService.objc/1.0 DroplrMac/2.0.5`

This is merely an example; what matters is that at least the application version should always be present on this header. Droplr's API servers have user agent blacklisting which will be used if erratic behavior is detected by a specific version of an application -- the server will reply with a message telling the client to upgrade to the latest version.

If the application does not update its `User-Agent` header according to its version, multiple versions may end up being blocked.

If you're develping an SDK that abstracts the network communication with Droplr's API Servers, your library should offer its users the possibility to append information about their application. For instance, `DroplrKit` always sends its name and version but requires each application to provide its own identification string.

`DroplrService.objc/1.0 %APP_IDENTIFIER%`


### Drop Creation Operations

Droplr's API server fully supports the 100-Continue header (RFC *TODO*). If possible, your client should implement this directive, as it'll save both time and bandwidth.

In short, The `100-Continue` is a value for the `Expect` header sent by the client when uploading data. When this header is set, the client will send the request headers and wait for a provisional response from the server (an `HTTP 100 Continue` response) before actually sending the data. This will enable the Droplr API Server to properly validate the request before receiving data and prepare to accomodate the incoming file.

Most modern HTTP implementations support this feature out of the box by simply setting the value `100-Continue` on the `Expects` header or via some other well-known request property.

When this feature is not used and the upload does not pass validation, it will be interrupted before all the data is sent through. This will likely cause problems with faulty HTTP implementations.


### Drop Privacy

Droplr supports three privacy modes, which reflects how drops are visible via their codes.

<aside class="notice">
  Drop privacy concerns only non-owner viewers of the drop so your application does not need any special handling when performing operations, such as <strong>Read drop</strong> or <strong>List drops</strong>. Drop privacy is used by Droplr's webapp to determine how to display (or how <em>not</em> to display) a given drop.
</aside>

No matter what the privacy mode is upon its creation, the drop will *always* have default values for all the fields mentioned in the sections below (**short code**, **obscure code** and **password**). Beware that older drops (pre-privacy era) may not include **password** field when retrieved so never assume password is always present.

#### Public

This is the default mode, reflected by the value `PUBLIC` in the `privacy` JSON field or the `x-droplr-privacy` Custom HTTP header.

Drops configured as `PUBLIC` are accessible either by their **short code** (e.g. `http://d.pr/xkcd`) or their **obscure code** (covered up next).

No special handling is required and apps may use the value of the shortlink field (`shortlink` JSON field or `x-droplr-shortlink`) in a drop directly.

**Short code** is an alphanumeric string that fits the pattern `[a-zA-Z0-9]+`.

#### Obscure

When a drop is configured to use its obscure code, it will only be accessible by its 16-char-long code, thus making it significantly harder to guess. A drop must be handled as obscure when the value `OBSCURE` is present in the `privacy` JSON field or the `x-droplr-privacy` Custom HTTP header.

Drops configured as `OBSCURE` are accessible *only* by their **obscure code** (e.g.: `http://d.pr/aF03GzuIqL0OeXsA`) and will return 404 (`Not found`) if someone tries to access them with their **short code**.

Since the server will automatically use the **obscure code** in the shortlink field (`shortlink` JSON field or `x-droplr-shortlink` Custom HTTP header) when returning drops, no special handling is required and applications may use this value directly.

**Obscure code** is an alphanumeric string that fits the pattern `[a-zA-Z0-9]{16}`.

#### Private

Drops configured as private can either be viewed by their **short code** or their **obscure code** but will always require a password to be viewed. A drop must be handled as private when the value `PRIVATE` is present in the `privacy` JSON field or the `x-droplr-privacy` Custom HTTP header.

When configured to private mode, the server will use the **short code** in the shortlink field (`shortlink` JSON field or `x-droplr-shortlink`) when returning drops, but in order for the shortlink to be directly accessibly (i.e. copy+paste accessible) you must append a forward-slash and the value of the password field (`password` JSON field or `x-droplr-password` Custom HTTP header).

For instance, a drop with **short code** `xkcd` and **password** `verySafePassword` would have its shortlink returned as `http://d.pr/xkcd` and would be directly accessible with `http://d.pr/xkcd/verySafePassword`.

<aside class="warning">
  Thumbnail shortlink modifiers (trailing `-`, `/thumbnail`, `/small` and `/medium`) as well as direct content access modifiers (trailing `+`) follow password-protected drop convetions too! If you were to fetch a thumbnail for an image with code `xkcd` and shortlink `http://d.pr/i/xkcd` by appending a trailing minus sign (`http://d.pr/i/xkcd-`), you would have to use `http://d.pr/i/xkcd/password-` instead.  
</aside>

If the password is not appended to the shortlink (thus creating an **accessible link**), the viewer will be presented with a screen requiring password input. This may be desired behavior if you wish to send the link for a drop via one transmission medium and the password via another.

**Password** is an alphanumeric string that fits the pattern `[a-zA-Z0-9]{4,32}`, although by default, an 8-char-long password is assigned. The character restriction for the password (alphanumeric chars only) is required to ensure that the drop is always accessible via a valid URL, since special chars would potentially break browser implementations and thus rendering the drop inaccessible.

### Accessible Drop URLs

```
String accessibleUrl(Drop drop) {
    switch (drop.privacy) {
        case "PRIVATE":
            return drop.shortlink + "/" + drop.password;

        case "PUBLIC":
        case "OBSCURE":
        case default:
            return drop.shortlink;
}
```
With any arbitrary language, assuming `Drop` represents a properly parsed drop response from the server, this algorithm will ensure that an accessible url is generated for a drop.
