# Private Operations

## Session-based Authentication

Droplr's API server notion of session is a bit different from what the name may imply. The basis of the authentication system is the email/password based authentication. However, this system is not adequate for the anonymous drag & drop upload feature presented by the Droplr's web site.

Droplr's API had, therefore, to support some kind of anonymous authentication mechanism that worked pretty much the same way as an email/password authentication system, except accounts could be created on-the-fly without registration info.

The session-based authentication works by performing calls to the server using a variant of the authentication mechanism and providing a unique identifier - the session identifier - instead of the typical email/password combination.

A session works pretty much the same way as a regular account, except it has more restrictive limits:

* Upload size is limited to 10MB per file
* The daily upload cap is 10 drops
* The maximum space allowed is 100MB
* After a period of 3 days of inactivity, all drops are deleted

The idea of anonymous upload is to allow people to try out Droplr really quick and then proceed to register a regular account. In the process of registering, all the previous anonymous uploads are migrated -- if a user uploads a file using the anonymous drag&drop and then registers, that file will be present when he lists the drops in his new account.

<aside class="notice">
  The session id must be 32-char long alphanumeric string (MD5 hash).
</aside>

### Session Authentication Formula

Starting from the end, an example of how an session Authorization header should look is:

`Authorization: droplrses cHVibGlja2V5OmQwNmY2ZTZlOTEyOGEyMzkzYjczNThmZjcwMTI0NTUw:QC0bWeSf8m979k4+AJ6lLxeMAfg=`

This can be decomposed as:

`Authorization: droplrses BASE64(ApplicationPublicKey:SessionId):signature`

Where `ApplicationPublicKey` and `SessionId` are the application's assigned public key and the session identifier. The formula to compute the `signature` parameter is:

`HMAC_SHA1(ApplicationPrivateKey:Password, stringToSign)`

The first big difference to the user based authentication system appears here. Since we're talking about anonymous volatile sessions, there are not passwords set. In order for Droplr to ensure the validity of the request, the password is itself a token calculated based on the session id. Its formula is thus:

`# generate an authenticity token`

`authenticityToken = MD5(ApplicationPrivateKey:SessionId:Salt)`

`# the password is the first half of the sessionId concatenated with the second half of the authenticity token`

`Password = SUBSTRING(SessionId, 0, 16) + SUBSTRING(authenticityToken, 16, 32)`

The `Salt` is a safe string (not documented here) to further reduce the chances of creating a valid request for abuse.

The process to create the `stringToSign` token is exactly the same as for user based authentication.

### Session Creation

The only way to create new sessions using session-based authentication is to perform a drop upload. When using session based authentication against `Create Drop - Link`, `Create Drop - Note` or `Create Drop - File` operations, the server will either associate the drop with an existing session or create a new record, if no previous session with the same id existed.


## Anonyomous Authentication

Droplr's API server also supports anonymous user authentication.
This authentication's formula is exactly the same as user-based authentication, except for the fact that the email must be `anonymous@droplr.com` and the password is a SHA-1 hash of the string `anonymous`.
Furthermore, the type of authentication identifier of the *Authorization* header is `droplranon` rather than `droplr`.

Example:

`Authorization: droplranon cHVibGlja2V5OmFub255bW91c0Bkcm9wbHIuY29t:m3n/LQOwt3Cv95KfJsnlbG2R/lM=`

The applications of this authentication are limited; currently it's only valid for the creation of new accounts and reading drops -- assuming the application has enough [privileges](#privileges) to execute the operations.


## Actions

### Create Account

> Example Request

```javascript
var user = JSON.stringify({
  "email": "test-26z84u30@droplr.com",
  "password": "5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8",
  "firstName": "Node",
  "lastName": "Client"
});

request({
   "method":"POST",
   "url":"http://sandbox.droplr.com:8069/account",
   "headers":{
      "Date":"Fri, 01 Aug 2014 19:51:54 GMT",
      "Accept":"application/json; version=0.9",
      "User-Agent":"droplr-node-client",
      "Content-Type":"application/json",
      "Authorization":"droplranon YXBwXzBfcHVibGlja2V5OmFub255bW91c0Bkcm9wbHIuY29t:gR0G6jV8LHGwbhoN0XFkI7TJu6k="
   },
   "body": user
}, function(err, resp, body) {
     // Do something with the response
});
```

> Example Response

```json
{
   "teamName":"droplr",
   "totalSpace":107374182400,
   "teamAdmin":true,
   "team":"id",
   "type":"PRO",
   "id":"id",
   "extraSpace":0,
   "email":"user@droplr.com",
   "createdAt":1406919673658,
   "verified":false,
   "dropPrivacy":"PRIVATE",
   "subdomain":"droplr",
   "trial":true,
   "subscriptionEnd":1409511673658
}
```

* **Description:** Create an account
* **URI:** `/account`
* **Method:** `POST`

##### Input Parameters

| Name | Type | Description |
| :--- | :--- | :---------- |
| email| string | **Required** The email of the account to be created |
| password | string | **Required** The account password, hashed with MD5 algorythm |
| domain | string | The domain to be used for shortlinks |
| domainType | string | What should be used in shortlinks DEFAULT, DOMAIN, SUB_DOMAIN|
| subdomain | string | The subdomain to be used shortlinks (e.g. subdomain.d.pr/xkdg). |
| firstName | string | The user's first name |
| lastName | string | The user's last name |
| username | string | The user's username |
| dropPrivacy | string | PUBLIC or PRIVATE |
| theme | string | light or dark |

##### Error Responses

##### Response if email or password are missing

`400 Bad Request`

Header Key | Header Value |
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidEmail`
`droplr-errordetails` | `Email is not valid`
`x-droplr-errorcode` | `CreateAccount.InvalidEmail`
`x-droplr-errordetails` | `Email is not valid`

##### Response if the email is not valid

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidEmail`
`droplr-errordetails` | `Email is not valid`
`x-droplr-errorcode` | `CreateAccount.InvalidEmail`
`x-droplr-errordetails` | `Email is not valid`

##### Response if the email is already taken

`409 Conflict`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.EmailAlreadyTaken`
`droplr-errordetails` | `That email is already taken; if it's yours please contact Droplr support`
`x-droplr-errorcode` | `CreateAccount.EmailAlreadyTaken`
`x-droplr-errordetails` | `That email is already taken; if it's yours please contact Droplr support`

##### Response if the password is not valid

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidPassword`
`droplr-errordetails` | `Password must be a SHA1 hash of the user input`
`x-droplr-errorcode` | `CreateAccount.InvalidPassword`
`x-droplr-errordetails` | `Password must be a SHA1 hash of the user input`

##### Response if the username is not valid

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidUsername`
`droplr-errordetails` | `Invalid Username`
`x-droplr-errorcode` | `CreateAccount.InvalidUsername`
`x-droplr-errordetails` | `Invalid Username`

##### Response if the username is already taken

`409 Conflict`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.UsernameAlreadyTaken`
`droplr-errordetails` | `That username is already taken`
`x-droplr-errorcode` | `CreateAccount.UsernameAlreadyTaken`
`x-droplr-errordetails` | `That username is already taken`

##### Response if domain is already taken

`409 Conflict`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.DomainAlreadyTaken`
`droplr-errordetails` | `Domain already taken`
`x-droplr-errorcode` | `CreateAccount.DomainAlreadyTaken`
`x-droplr-errordetails` | `Domain already taken`


##### Notes

* The authentication used for this action must be Anonymous User authentication. Only first party applications are allowed to create accounts.

* Subdomain will be used for teamName.

* If a subdomain is provided but is in use, an incremental integer will be added as a suffix.

* If a subdomain is not provided one will be derived from the email domain.


### Create Team Member Account

> Example Request

```javascript
var user = JSON.stringify({
  "email": "test-26z84u30@droplr.com",
  "password": "5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8",
  "firstName": "Node",
  "lastName": "Client"
});

request({
   "method":"POST",
   "url":"http://sandbox.droplr.com:8069/teams/1/account",
   "headers":{
      "Date":"Fri, 01 Aug 2014 19:51:54 GMT",
      "Accept":"application/json; version=0.9",
      "User-Agent":"droplr-node-client",
      "Content-Type":"application/json",
      "Authorization":"droplranon YXBwXzBfcHVibGlja2V5OmFub255bW91c0Bkcm9wbHIuY29t:gR0G6jV8LHGwbhoN0XFkI7TJu6k="
   },
   "body": user
}, function(err, resp, body) {
     // Do something with the response
});
```

> Example Response

```json
{  
   "teamName":"my co",
   "useLogo":true,
   "logo":"http://logo",
   "totalSpace":107374182400,
   "usedSpace":0,
   "theme":"light",
   "team":"team_id",
   "domainType":"DEFAULT",
   "type":"PRO",
   "id":"id",
   "dropCount":0,
   "maxUploadSize":2147483648,
   "rootRedirect":"http://biasedbit.com",
   "username":"user1",
   "extraSpace":0,
   "email":"user1@droplr.com",
   "createdAt":1406929260072,
   "verified":false,
   "dropPrivacy":"PUBLIC",
   "subdomain":"myco",
   "domain":"drops.biasedbit.com",
   "useRootRedirect":true,
   "activeDrops":0,
   "subscriptionEnd":1406929258896
}
```

* **Description:** Create team member account
* **URI:** `/teams/:team_id/account`
* **Method:** `POST`


##### Input Parameters

| Name | Type | Description |
| :--- | :--- | :---------- |
| email| string | **Required** The email of the account to be created |
| password | string | **Required** The account password, hashed with MD5 algorythm |
| firstName | string | The user's first name |
| lastName | string | The user's last name |
| username | string | The user's username |

##### Error Responses

##### Response if email or password are missing

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.MissingEmailOrPassword`
`droplr-errordetails` | `Missing email and/or password`
`x-droplr-errorcode` | `CreateAccount.MissingEmailOrPassword`
`x-droplr-errordetails` | `Missing email and/or password`

##### Response if the email is not valid

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidEmail`
`droplr-errordetails` | `Email is not valid`
`x-droplr-errorcode` | `CreateAccount.InvalidEmail`
`x-droplr-errordetails` | `Email is not valid`

##### Response if the email is already taken

`409 Conflict`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.EmailAlreadyTaken`
`droplr-errordetails` | `That email is already taken; if it's yours please contact Droplr support`
`x-droplr-errorcode` | `CreateAccount.EmailAlreadyTaken`
`x-droplr-errordetails` | `That email is already taken; if it's yours please contact Droplr support`

##### Response if the password is not valid

`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidPassword`
`droplr-errordetails` | `Password must be a SHA1 hash of the user input`
`x-droplr-errorcode` | `CreateAccount.InvalidPassword`
`x-droplr-errordetails` | `Password must be a SHA1 hash of the user input`

##### Response if the username is not valid


`400 Bad Request`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` | `CreateAccount.InvalidUsername`
`droplr-errordetails` | `Invalid Username`
`x-droplr-errorcode` | `CreateAccount.InvalidUsername`
`x-droplr-errordetails` | `Invalid Username`

##### Response if the username is already taken

`409 Conflict`

Header Key | Header Value
---------- | ------------ |
`droplr-errorcode` |` CreateAccount.UsernameAlreadyTaken`
`droplr-errordetails` |` That username is already taken`
`x-droplr-errorcode` |` CreateAccount.UsernameAlreadyTaken`
`x-droplr-errordetails` |` That username is already taken`


##### Notes

* The authentication used for this action must be Anonymous User authentication. Only first party applications are allowed to create accounts.


### Update a Customer ID

> Example Request

```javascript
var subscriptionId = JSON.stringify({
  "email": "test-26z84u30@droplr.com",
  "service": "stripe",
  "customerId": "xkdc"
});

request({  
   "method":"PUT",
   "url":"http://sandbox.droplr.com:8069/account/customer_id",
   "headers":{  
      "Date":"Wed, 06 Aug 2014 20:55:35 GMT",
      "Accept":"application/json; version=1.0",
      "User-Agent":"droplr-node-client",
      "Content-Type":"application/json; v=1.0; encoding=utf-8;",
      "Authorization":"droplranon YXBwXzBfcHVibGlja2V5OmFub255bW91c0Bkcm9wbHIuY29t:/xAaACyWDdtVIdCMrfRU7xos9TY="
   },
   "body": subscriptionId
}, function(err, resp, body) {
     // Do something with the response
});
```

* **Description:** Create a customer_id
* **URI:** `/account/customer_id`
* **Method:** `PUT`

##### Input Parameters

| Name | Type | Description |
| :--- | :--- | :---------- |
| email| string | **Required** The email of the account to be update |
| service | string | **Required** The service for which customerId will be updated |
| customerId | string | **Required** The customerId to be set for service |


##### Error Responses


### Convert Session to Account

* **Description:** Create session to account
* **URI:** `/sessions/convert`
* **Method:** `POST`

##### Input Parameters

Same input parameters and format as [create account](#create-account) action.

##### Responses
Same output parameters and format as [create account](#create-account) action.


##### Notes

* The authentication used for this action must be session authentication. Only first party web application tier apps can perform this action (basically just the website at droplr.com).

* This action creates a new account with the provided details and migrates all the drops from the session - which is invalidated - to the new user account.
