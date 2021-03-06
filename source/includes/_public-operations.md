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

#### Common Error Responses

There are a great deal of errors that are common to all operations. The following is a comprehensive list of these errors along with the matching HTTP status code, a description of the cause and the message that is sent in the `x-droplr-errordetails` header.

<!-- Request.BlacklistedUserAgent | 403 | The client user agent was blacklisted. Blacklisting user agents is an effective way to block a specific version of an application without deleting its access credentials. Your application may be blacklisted if erratic behavior is detected. Since we assume this is a temporary issue, we convey an error message that warns users to update their application to the latest version. | Please update to the newest version -->

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**Internal.DataAccessError** | 503 | Temporary failure on Droplr's data storage system. | Temporary data access failure when performing operation
**Internal.TooManyRequest** | 503 | Server is under extremely heavy load and will discard some requests in order to keep serving a percentage of those. | Server is under heavy load; please try again later
**Maintenance.GoingDown** | 503 | Server is under maintenance and will be shutdown soon. | Server will be going down for maintenance shortly
**Maintenance.ReadOnly** | 503 | Server is under maintenance and only non-idempotent operations (read-only) will be allowed. | Server is under maintenance; only read operations will be allowed
**Authentication.InvalidAuthHeader** | 401 | Auth header is invalid; please refer to the Auth & auth guide. | Authorization header format is not in conformity with specification
**Authentication.UnknownScheme** | 401 | Authentication scheme is unknown; please refer to the Auth & auth guide. | Authentication scheme not supported: `%SCHEME%`
**Authentication.InvalidScheme** | 401 | The authentication scheme used was not valid for the requested operation. | Authentication scheme not supported for this action
**Authentication.InvalidSignature** | 401 | The HMAC-SHA1 signature does not match expected value. | HMAC SHA1 signature is invalid
**Authentication.ReplayedSignature** | 401 | The signature generated for the request has recently been used. Make sure the value of the `Date` header is being set according to the [auth](#authentication-amp-authorization) guide. | Signature has already been used
**Authentication.ClockSkew** | 401 | The client's clock is too far ahead/behind the server's clock. | Date in request (`%REQUEST_DATE%`) is too far ahead/behind the server date (`%SERVER_DATE%`)
**Authentication.UnknownApplication** | 401 | Application does not have credentials to perform operations on the server. | No such application
**Authentication.UnknownUser** | 401 | User with given identifier does not exist. | No such user
**Authentication.SignatureMismatch** | 401 | Signature is according to spec but result does not match expected. Typically means wrong password. | Invalid password
**Authorization.NoPrivileges** | 403 | Application attempted to execute an operation to which it does not have clearance. | This application has no permissions to execute this action
**Request.InvalidUri** | 400 | Malformed uri and/or query parameters on HTTP request line. | Invalid uri and/or query params
**Request.NoAction** | 404 | No operation matching request URI. | No action at the requested uri
**Request.BodyMustBeEmpty** | 400 | The application sent body for a request that doesn't require body. Server will immediately reply with error and close the connection to avoid unnecessary resource usage. | Request body must be empty
**Request.NoContentLength** | 400 | Client application sent content without specifying a `Content-Lenght` header. | This server always requires Content-Length header, even for chunked requests
**Request.ContentTooLarge** | 400 | Client application specified a value that exceeds the acceptable limits (2GB). | Content-Length indicates illegal size (over 2GB)
**Request.NoContentType** | 400 | Request did not bear a `Content-Type` header, which is mandatory when the request includes a body. | Content-Type header is mandatory
**Request.ContentLengthMismatch** | 400 | Announced content length (on `Content-Length` header) doesn't match actual content length. | Mismatch between announced content length and actual readable data
**Request.NoDateHeader** | 400 | No `Date` (or `x-droplr-date`) header found on request. This header is always mandatory. | No Date header found in request
**Request.NoAuthorizationHeader** | 400 | No `Authorization` header found on request. This header is always mandatory. | No Authorization header found in request
**Request.InvalidJson** | 400 | JSON content in request body could not be correctly decoded. | Failed to parse JSON request body
**Request.UnsupportedDataFormat** | 400 | Client application tried to perform an operation using an invalid data format (e.g. .xml). | Unsupported request data format: `%DATA_FORMAT%`
**Request.NoJson** | 400 | Server was expecting JSON content in request body but no content (or non-JSON content) was found. | A JSON body is required, along with content Content-Type header set
**Request.BadContentType** | 400 | Value of the header `Content-Type` is not a valid mime type. | Unable to parse Content-Type header value: `%CONTENT_TYPE%`
**Request.ChunksNotAcceptable** | 400 | Chunked transfer not acceptable for the requested operation. | Chunked requests not accepted for this action
**Request.PipelineAbuse** | 400 | Client application exceeded maximum allowed pipelined requests for a single connection. | Too many pipelined requests
**Request.BlacklistedUserAgent** | 403 | The client user agent was blacklisted. Blacklisting user agents is an effective way to block a specific version of an application without deleting its access credentials. Your application may be blacklisted if erratic behavior is detected. Since we assume this is a temporary issue, we convey an error message that warns users to update their application to the latest version. | Please update to the newest version

The following errors apply only to drop creation operations (shorten link, create note and upload file).

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**CreateDrop.TooManyUploads** | 503 | Too many concurrent uploads and the server needs to reject some in order to maintain an acceptable bandwidth for all uploads. | Cannot process your request at this time, please try again later
**CreateDrop.MaxSizeExceeded** | 400 | The maximum upload size was exceeded. Drops have a size limit based on type and the user's account | Max upload size limit exceeded: `%SIZE%`
**CreateDrop.NoSpace** | 507 |The user has hit the limits of his or her account and cannot upload files until he clears out some space. | Used `%USED%` of available `%AVAILABLE%`
**CreateDrop.ContentTypeMustMatch** | 400 | The value in `Content-Type` header does not match any of the values in the content type white list. | Content-Type header is mandatory and must match `%LIST%`
**CreateDrop.InvalidPrivacy** | 400 | The value in `privacy` query param or `x-droplr-privacy` custom header is not a valid privacy value. | Invalid privacy value
**CreateDrop.InvalidPrivacy** | 400 | The value in `password` query param or `x-droplr-password` custom header is not a valid password value; it either has invalid length (too short or too long) or contains invalid characters. | Invalid password value


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
  Thumbnail shortlink modifiers (trailing <code>-</code>, <code>/thumbnail</code>, <code>/small</code> and <code>/medium</code>) as well as direct content access modifiers (trailing <code>+</code>) follow password-protected drop convetions too! If you were to fetch a thumbnail for an image with code <code>xkcd</code> and shortlink <code>http://d.pr/i/xkcd</code> by appending a trailing minus sign (<code>http://d.pr/i/xkcd-</code>), you would have to use <code>http://d.pr/i/xkcd/password-</code> instead.  
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

## Actions

### Shorten Link

> Example Request

```shell
POST /links HTTP/1.0
...
x-droplr-privacy: PRIVATE
x-droplr-password: quagmire
...
```

> Example Response

```
HTTP/1.1 201 Created
x-droplr-code: xkcd
x-droplr-createdat: 1304642398598
x-droplr-type: LINK
x-droplr-title: QSBmaW5lIG5vdGU=
x-droplr-size: 11
x-droplr-usedspace: 11
x-droplr-shortlink: http://d.pr/xkcd
x-droplr-totalspace: 10000
Content-Length: 0

HTTP/1.1 201 Created
Content-Type: application/json; encoding=utf-8
Content-Length: 211
```
```json
{
  "totalSpace":10000,
  "title":"http://droplr.com",
  "usedSpace":11,
  "createdAt":1304642741028,
  "code":"xkcd",
  "type":"LINK",
  "shortlink":"http://d.pr/xkcd",
  "size":11,
  "privacy":"PUBLIC",
  "obscureCode":"xkcdXKCDxkcdXKCD"
}
```


Create a short link to a long link.

**Supported formats:** HEADERS, JSON


##### Input Parameters

`POST /links`

Parameter | Description | Header Key | Query Parameter
--------- | ----------- | ---------- | --------------- |
**Drop privacy** | The privacy with which this drop should be created, overriding default account settings. | `x-droplr-privacy` | `privacy`
**Drop password** | The password with which this drop should be created, overriding default password generation. | `x-droplr-password` | `password`


##### Output Parameters

Parameter | Description | Header Key | JSON Field
--------- | ----------- | ---------- | ---------- |
**Drop code** | The code for the drop. | `x-droplr-code` | `code`
**Created at** | The timestamp, since UTC (in milliseconds), when the drop was created. | `x-droplr-createdat` | `createdAt`
**Type** | The type of drop (LINK, NOTE, IMAGE, VIDEO, AUDIO, FILE). | `x-droplr-type` | `type`
**Title** | The title of the drop. | `x-droplr-title` header -- Base64 encoded (in order to preserve UTF-8 char | `title`
**Size** | The size of the drop, in bytes. | `x-droplr-size` | `size`
**Privacy** | The privacy mode with which the drop was created. Possible values are `PUBLIC`, `PRIVATE` or `OBSCURE`. | `x-droplr-privacy` | `privacy`
**Password** | The password with which the drop was created. Will only be present of drop privacy was set to `PRIVATE`. Password is sent as plain text (reminder: the HTTP connection is encrypted) as drop passwords can only contain characters allowed in a URL path component | `x-droplr-password` | `privacy`
**Obscure code** | A hard-to-guess 16 char unique string that maps to this drop. Drops can be accessed either via short code or obscure code. Drops configured to use `OBSCURE` privacy can only be accessed via obscure code. | `x-droplr-obscurecode` | `obscureCode`
**Shortlink** | The shortlink where the drop is available at; it's a concatenation of the user's custom domain (if set) or Droplr's default domain and the drop code. | `x-droplr-shortlink` | `shortlink`
**Used space** | The total used space for the account creating the drop, in bytes. | `x-droplr-usedspace` | `usedSpace`
**Total space** | The total space availabe for the account creating the drop, in bytes. | `x-droplr-totalspace` | `totalSpace`


##### Errors

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**CreateDrop.InvalidUrl** | 400 | The link to shorten is an invalid link. | Invalid URL
**CreateDrop.RecursiveLink** | 400 | The link to shorten is a link to one of Droplr's domains (or subdomains). | Cannot shorten links to Droplr's urls


##### Notes

* The title for the drop generated with this operation will be the original link itself. If the target is a webpage, the title will eventually be updated the the title of the webpage -- assuming there is one.
* This action does not allow shortening links to Droplr's domains (`droplr.com` and `d.pr`) or its subdomains.

### Create Note

> Example Request

```
POST /notes HTTP/1.1
...
Content-Type: text/plain
...

POST /notes HTTP/1.1
...
Content-Type: text/code
x-droplr-privacy: PRIVATE
x-droplr-password: quahog
...
```

> Example Response

```
HTTP/1.1 201 Created
x-droplr-code: xkcd
x-droplr-createdat: 1304642398598
x-droplr-type: NOTE
x-droplr-title: QSBmaW5lIG5vdGU=
x-droplr-size: 11
x-droplr-usedspace: 11
x-droplr-shortlink: http://d.pr/xkcd
x-droplr-totalspace: 10000
Content-Length: 0

HTTP/1.1 201 Created
Content-Type: application/json; encoding=utf-8
Content-Length: 211
```
```json
{
  "totalSpace":10000,
  "title":"http://droplr.com",
  "usedSpace":11,
  "createdAt":1304642741028,
  "code":"xkcd",
  "type":"NOTE",
  "shortlink":"http://d.pr/xkcd",
  "size":11,
  "privacy":"PUBLIC",
  "obscureCode":"xkcdXKCDxkcdXKCD"
}
```


Create a text based document and a return a short link for it

**Supported formats:** HEADERS, JSON
**Variants:** Plain text, Markdown, Textile, Code, etc


##### Input Parameters

`POST /notes`

Parameter | Description | 
--------- | ----------- | 
**Type of note** | The type of note is defined by setting the `Content-Type` header to `text/plain`, `text/markdown`, `text/textile` or `text/code`. As long as the `Content-Type` header is set to a mime type with main type being "text", the drop will be recognized as a note. The mime subtype is merely a hint so that applications viewing this drop can apply text formatting and/or styling (such as code markup or markdown-to-html rendering).

All other input parameters are the same as the [shorten link](#shorten-link) action.


##### Output Parameters

Parameter | Description | Header Key | JSON Field
--------- | ----------- | ---------- | ---------- |
**Variant** | The variant is a hint to applications on how to display the drop's content and/or its icon. Beware that this is basically a free-form field and can be set to any value by other applications. | `x-droplr-variant` | `variant`

All other output parameters are the same as the [shorten link](#shorten-link) action.


##### Notes

* The title of the drop will be based on the contents of the note; its first 100 characters with newlines replaced by spaces.


### Upload File

>Example Request

```
POST /files HTTP/1.1
...
x-droplr-filename: QSBmaW5lIG5vdGU=
...

POST /files HTTP/1.1
...
x-droplr-filename: QSBmaW5lIG5vdGU=
x-droplr-privacy: PRIVATE
x-droplr-password: wekapaw
...
```

> Example Response

```
HTTP/1.1 201 Created
x-droplr-code: xkcd
x-droplr-createdat: 1304642398598
x-droplr-type: FILE
x-droplr-title: QSBmaW5lIG5vdGU=
x-droplr-size: 11
x-droplr-usedspace: 11
x-droplr-shortlink: http://d.pr/xkcd
x-droplr-totalspace: 10000
Content-Length: 0

HTTP/1.1 201 Created
Content-Type: application/json; encoding=utf-8
Content-Length: 211
```
```json
{
  "totalSpace":10000,
  "title":"http://droplr.com",
  "usedSpace":11,
  "createdAt":1304642741028,
  "code":"xkcd",
  "type":"FILE",
  "shortlink":"http://d.pr/xkcd",
  "size":11,
  "privacy":"PUBLIC",
  "obscureCode":"xkcdXKCDxkcdXKCD"
}
```


Upload a file and return a short link for its location

**Supported formats:** HEADERS, JSON
**Variants:** Image, Audio, Video, File


##### Input Parameters

`POST /files`

Parameter | Description | Header Key | Query Parameter
--------- | ----------- | ---------- | --------------- |
**Filename** | The name under which the file should be stored. This property is always mandatory and can either be sent with query params or custom headers (query params take precedence). It must be sent as a query param or a header since the body of the request is reserved for the upload data. | `x-droplr-filename` | `filename`

All other input parameters are the same as the [create note](#create-note) action.


##### Output Parameters

All output parameters are the same as the [create note](#create-note) action.


##### Errors

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**CreateDrop.RemoteStorageOperationFailed** | 500 | There was an error storing the file's contents. | Error storing drop contents
**CreateDrop.MissingFilename** | 400 | Request did not bear a `x-droplr-filename` header with the name of the file. | Missing filename; please set query parameter or use custom header
**CreateDrop.FilenameNotBase64** | 400 | The server could not decode the contents of the header `x-droplr-filename` as Base 64. | Filename header must be base64 encoded


##### Notes

* The contents of the `x-droplr-filename` header **MUST NOT** be URL escaped and **MUST** be Base64 encoded. Since HTTP headers must be in US ASCII encoding, UTF-8 characters are not supported. Encoding UTF-8 text to Base64 (and then decoding at the server side) ensures that UTF-8 characters are supported as the drop's title.
* The filename header is always mandatory, whether the action is executed with JSON or HEADERS data format.

### Read Drop

>Example Request

```
GET /drops/xkcd HTTP/1.1

```

> Example Response

```
HTTP/1.1 201 Created
x-droplr-code: xkcd
x-droplr-createdat: 1304642398598
x-droplr-type: FILE
x-droplr-title: QSBmaW5lIG5vdGU=
x-droplr-size: 11
x-droplr-shortlink: http://d.pr/xkcd
Content-Length: 0

HTTP/1.1 201 Created
Content-Type: application/json; encoding=utf-8
Content-Length: 211
```
```json
{
  "title":"http://droplr.com",
  "createdAt":1304642741028,
  "code":"xkcd",
  "type":"FILE",
  "shortlink":"http://d.pr/xkcd",
  "size":11,
  "privacy":"PUBLIC",
  "obscureCode":"xkcdXKCDxkcdXKCD"
}
```

Read the contents of a previously created drop

**Supported formats:** HEADERS, JSON


##### Input Parameters

`GET /drops/:code`

The only input parameter is the drop `code` supplied in the URL; e.g. `GET /drops/xkcd`.


##### Output Parameters

All output parameters are the same as [upload file](#upload-file) except for `usedSpace` and `totalSpace`, which are not present in the response to this request.


##### Errors

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**ReadDrop.NoDrop** | 404 | Request specified a drop code that does not exist. | No such drop
**ReadDrop.NotOwner** | 403 | Request specified a drop code that is not owned by the user whose credentials were used to sign the request. | You're not the other of that drop


##### Notes

* Drops can only be read by their creators. If you request details on a drop not owned by the credentials with which you sign the request, the server will return an unauthorized response.


### List Drops

>Example Request

```json
{
  "startIndex":10,
  "amount":10,
  "type":"link"
}

{
  "since":1299648278321
}

{
  "until":1299648278321,
  "sortBy":"VIEWS",
  "order":"ASC"
}
```

>Example Response

```
HTTP/1.1 200 OK
Content-Type: application/json; encoding=utf-8
Content-Length: ...
```

```json
[
  {
    "content":"http://redis.io",
    "title":"http://redis.io",
    "obscureCode":"POxqReCZlYk14iN6",
    "views":0,
    "createdAt":1333505121880,
    "privacy":"PUBLIC",
    "lastAccess":1333505121880,
    "code":"xkcd",
    "type":"LINK",
    "shortlink":"http://d.pr/xkcd",
    "size":15
  },
  {
    "content":"a very long note that will be truncate...",
    "title":"a note",
    "obscureCode":"XslQ13B12Qk14DN6",
    "views":14,
    "createdAt":1333505161880,
    "privacy":"PRIVATE",
    "password":"lePass",
    "lastAccess":1333505121880,
    "code":"xkce",
    "type":"NOTE",
    "variant":"plain",
    "shortlink":"http://d.pr/xkce",
    "size":1500
  }
]
```


Retrieve a list of previously created drops with the possibility to combine filters

**Supported formats:** JSON


##### Input Parameters

`GET /drops`

Parameter | Description | Header Key | Query Parameter
--------- | ----------- | ---------- | --------------- |
**Offset (optional, defaults to 0)** | The drop index at which the list should start. Useful for pagination, the meaning of index zero depends on the sort order (which defaults to latest-drops-first). See the **Order** parameter. | `offset` | `offset`
**Amount (optional, defaults to 10)** | The amount of drops to retrieve. This value will be capped by the server if too many drops are requested. | `amount` | `amount`
**Type (optional, defaults to ALL)** | Filter by type of drop. | `type` | `type`
**Sort by (optional, defaults to `CREATION`)** | Sort the retrieved drops by this field. | `sortBy` | `sortBy`
**Order (optional, defaults to `DESC`)** | The order by which drops should be retrieved. | `order` | `order`
**Since timestamp (optional)** | List all drops created after a given timestamp, in milliseconds elapsed since UTC. | `since` | `since`
**Until timestamp (optional)** | List all drops created until a given timestamp, in milliseconds elapsed since UTC. | `until` | `until`


##### Output Parameters

A list with drop information. Each entry in this list has the same format and parameters as the [read drop](#read-drop) action, except that the content for notes will be truncated (to avoid very large data transfers).


##### Notes

* This action will only return drops created by the user account whose credentials were used to authenticate the request.
* Acceptable values for **Type** input parameter are `LINK`, `NOTE`, `IMAGE`, `AUDIO`, `VIDEO` and `FILE`.
* Acceptable values for the **Sort by** criteria are `CREATION` (default), `CODE`, `TITLE`, `SIZE`, `ACTIVITY` and `VIEWS`.
* Acceptable values for the **Sort order** criterial are `ASC` (for ascending) and `DESC` (for descending).


### Delete Drop

>Example Request

```
DELETE /drops/xkcd HTTP/1.1

```


Delete a previously created drop

**Supported formats:** HEADERS (there is no output, so defaults to HEADERS)


##### Input Parameters

`DELETE /drops/:code`

The only input parameter is the drop `code` supplied in the URL; e.g. `DELETE /drops/xkcd`.


##### Output Parameters

N/A


##### Errors

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**DeleteDrop.NoDrop** | 404 | Drop for specified code does not exist. | No such drop
**DeleteDrop.NotOwner** | 403 | Request specified a drop for deletion that is not owned by the user whose credentials were used to sign the request. | You're not the owner of that drop
**DeleteDrop.AlreadyDeleted** | 410 | Request specified a drop for deletion that has already been deleted. | Record already deleted


##### Notes

* This action will only delete drops created by the user account whose credentials were used to authenticate the request.


### Read Account Details

>Example Request

```
GET /account HTTP/1.1
```

> Example Response

```json
{
  "key":"value"
}
```


Retrieve information of a given account

**Supported formats:** HEADERS, JSON


##### Input Parameters

`GET /account`


##### Output Parameters

Parameter | Description | Header Key | JSON Field
--------- | ----------- | ---------- | ---------- |
**User id** | The unique user id assigned to this account. | `x-droplr-id` | `id` 
**Created at** | The POSIX timestamp (in milliseconds) when this account was created. | `x-droplr-createdat` | `createdAt`
**Type** | The type of account (PRO, REGULAR). | `x-droplr-type` | `type`
**Subscription end** | The exact date of the Pro subscription expiration, in milliseconds elapsed since UTC. This value will be present if **Type** is `PRO`. | `x-droplr-subscriptionend` | `subscriptionEnd`
**Max upload size** | The maximum size for file uploads, depends on user account. Applications should use this value to test whether an upload can be performed before performing the upload. | `x-droplr-maxuploadsize` | `uploadSize`
**Used space** | The total used space by this user. | `x-droplr-usedspace` | `usedSpace`
**Email** | The user's current email. | `x-droplr-email` | `email`
**Use custom domain** | A flag that indicates what type of domain should be used (if **Custom domain** property is set). | `x-droplr-domaintype` | `domainType`
**Custom domain** | A custom domain to use when returning shortlinks for drops, if the **Use custom domain** flag is set. | `x-droplr-domain` | `domain`
**Use root redirect** | A flag that indicates whether root redirects should be used when hitting the custom domain without a drop code. This property only makes sense if both **Custom domain** and **Root redirect** properties are set. | `x-droplr-userootredirect` | `useRootRedirect`
**Root redirect** | Where to redirect HTTP requests to the custom domain (**Custom domain** property). | `x-droplr-rootredirect` | `rootRedirect`
**Drop privacy** | The privacy setting with which new drops will be created. Acceptable values are `PUBLIC`, `PRIVATE` and `OBSCURE`. | `x-droplr-dropprivacy` | `dropPrivacy`
**Theme** | The theme to use on Droplr's website. This is a free-form string but typical values are 'default', 'light' and 'dark'. If the value is not recognized, the default theme will be used. Your application can also rely on this value to change its appearance. | `x-droplr-theme` | `theme`
**Drop count** | The number of drops created by this user. | `x-droplr-dropcount` | `dropCount` 


##### Notes

* The account that's read is the one whose credentials are provided in the authentication header.


### Edit Account Details

>Example Request

```
PUT /account HTTP/1.1
```

```json
  {
    "password":"5f4dcc3b5aa765d61d8327deb882cf99",
    "domain":"d.pr"
  }
```

> Example Response

```json
{
  "key":"value"
}
```


Edit information for an account

**Supported formats:** HEADERS, JSON


##### Input Parameters

`PUT /account`

Parameter | Description | Header Key | Query Parameter
--------- | ----------- | ---------- | --------------- |
**Password** | The user's new password, hashed with SHA-1 algorithm. | `x-droplr-password` | `password`
**Theme** | The theme to use on Droplr's website. This is a free-form string but typical values are 'default', 'light' and 'dark'. If the value is not recognized, the default theme will be used. Your application can also rely on this value to change its appearance. | `x-droplr-theme` | `theme`
**Use custom domain** | A flag that indicates what type of domain should be used (if **Custom domain** property is set). | `x-droplr-domaintype` | `domainType`
**Custom domain** | A custom domain to use when returning shortlinks for drops, if the **Use custom domain** flag is set. | `x-droplr-domain` | `domain`
**Use root redirect** | A flag that indicates whether root redirects should be used when hitting the custom domain without a drop code. This property only makes sense if both **Custom domain** and **Root redirect** properties are set. | `x-droplr-userootredirect` | `useRootRedirect`
**Root redirect** | Where to redirect HTTP requests to the custom domain (**Custom domain** property). | `x-droplr-rootredirect` | `rootRedirect`
**Drop privacy** | The privacy setting with which new drops will be created. Acceptable values are `PUBLIC`, `PRIVATE` and `OBSCURE`. To change the privacy to `PRIVATE` a Pro account is required. All other changes are allowed. | `x-droplr-dropprivacy` | `dropPrivacy`


##### Output Parameters

All output parameters are the same as the [read account details](#read-account-details) action.


##### Errors

Error Code | Status Code | Cause | Error Message
---------- | ----------- | ----- | ------------- |
**EditAccount.NoUpdateData** | 400 | No update data was found on request (no-op request). | No update data
**EditAccount.NoUpdateData** | 400 | No update data was found on request (no-op request). | No update data
**EditAccount.ProRequired** | 403 | The operation attempted to edit one or more fields for which a Pro subscription was required. | A Pro subscription is required to edit field `%FIELD%`
**EditAccount.ProRequiredEnableDomain** | 403 | The operation attempted to edit custom domain usage with a non-Pro user. | A Pro subscription is required to enable custom domain usage
**EditAccount.ProRequiredEnableRootRedirect** | 403 | The operation attempted to edit root redirect usage with a non-Pro user. | A Pro subscription is required to enable root redirect usage
**EditAccount.InvalidDomain** | 400 | The domain specified for the custom domain field is not valid (e.g. includes `http(s)://` or other invalid characters). | Domain is invalid
**EditAccount.InvalidRootRedirect** | 400 | The input password parameter is not a SHA-1 hash. | Password must be a SHA1 hash of the user input
**EditAccount.InvalidPassword** | 400 | The root redirect specified is not a valid URL. | Root redirect is invalid
**EditAccount.Deleted** | 400 | The account has either been locked or deleted. | Account has been deleted


##### Notes

* Only Pro accounts can change their custom domain, root redirect and privacy to `PUBLIC`. Regular accounts can change privacy option, as long as target privacy option is `PUBLIC` or `OBSCURE`.
* The **domain** input parameter must **not** contain the scheme information, i.e. to use the custom domain `http://droplr.biasedbit.com`, the app must only provide the string `droplr.biasedbit.com`.
