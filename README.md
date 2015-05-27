# HTTP HMAC Spec

This spec documents an HMAC message format for securing RESTful web APIs.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

## Spec

Production services implementing this spec must only accept requests using HTTPS.

## Overview of Header and Signature

The pseudocode below illustrates construction of the HTTP "Authorization" header and signature:

```
Authorization: acquia-http-hmac realm="Example",
               id="identifier",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               headers="",
               signature="Signature"

X-Acquia-Timestamp: 1432075982

Signature = Base64( HMAC( SecretKey, Signature-Base-String ) );

Signature-Base-String =
    HTTP-Verb + "\n" +
    host  + "\n" +
    Path + "\n" +
    Query-Parameters + "\n" +
    Header-Parameters + "\n" +
    Added-Signed-Headers (if any) + "\n" +
    Timestamp + (except for GET/HEAD) "\n" +
    Content-Type + "\n" +
    Body-Hash
;
```

`"\n"` denotes a Unix-style line feed (ASCII code `0x0A`).

#### Secret Key

The secret key should be a 256 to 512 bit binary value. The secret key should be stored as a base64-encoded
string representation and decoded to binary before use.

#### Authorization Header

The value of the `Authorization` header contains the following parts:

* `realm`: The provider, for example "Acquia", "MyCompany", etc.
* `id`: The API key's unique identifier, which is an arbitrary string
* `nonce`:  a hex version 4 (or version 1) [UUID](http://tools.ietf.org/rfc/rfc4122.txt).
* `version`: the version of this spec
* `headers`: a list of additional request headers that are to be included in the signature base string. These are lower-case, and separated with ;
* `signature`: the Signature (base64 encoded) as described below.

Each value should be enclosed in double quotes and urlencoded (percent encoded).

Note that the name of this (standard) header is misleading - it carries authentication information.

#### X-Acquia-Timestamp Header

A Unix timestamp (integer seconds since Jan 1, 1970 UTC). Required for all requests. If this value differs by more than 900 seconds (15 minutes) from the time of the server, the request will be rejected.

#### X-Acquia-Content-SHA256 Header

The base64 encoded SHA-256 hash value used to generate the signature base string. This is analogous to the standard Content-MD5 header. Required for any request except GET or HEAD.

#### Signature

The signature is a base64 encoded binary HMAC digest generated from the
following parts:

* `HMAC`: HMAC-sha256 algorithm
* `SecretKey`: The API key's shared secret
* `Signature Base String`: The string being signed as described below

#### Signature Base String

The signature base string is a concatenated string generated from the following parts:

* `HTTP-Verb`: The uppercase HTTP request method e.g. "GET", "POST"
*  The (lowercase) hostname, matching the HTTP "Host" request header field
* `Path`: The HTTP request path + query string, e.g. `/resource/11`
* `Parameters`: normalized parameters similar to section 9.1.1 of OAuth 1.0a.  Any query parameters or empty string.  Parameters are sorted by name and separated by '&' with name and value separated by =, percent encoded (urlencoded).
* `Header-Parameters`: normalized parameters similar to section 9.1.1 of OAuth 1.0a.  The parameters are the id, nonce, realm, and version from the Authorization header. Parameters are sorted by name and separated by '&' with name and value separated by =, percent encoded (urlencoded)
* `Added Signed Headers`: The normalized header names and values specified in the headers parameter of the Authorization header. Names should be lower-cased, sorted by name, separated from value by a colon and the value followed by a newline so each extra header is on its own line.
* `Timestamp`:  The value of the X-Acquia-Timestamp header
* `Content-Type`: The lowercase value of the "Content-type" header (or empty string if absent). Omit for a GET or HEAD request.
* `Body-Hash`: SHA-256 digest of the raw body of the HTTP request, for POST, PUT, PATCH, DELETE or other requests that may have a body. Omit for GET or HEAD.

#### GET Example

https://example.acquiapipet.net/v1.0/task-status/133?limit=10

Assuming the client ID is efdde334-fe7b-11e4-a322-1697f925ec7b and base64 encoded secret key is W5PeGMxSItNerkNFqQMfYiJvH14WzVJMy54CPoTAYoI=

Authorization header =
```
Authorization: acquia-http-hmac realm="Pipet%20service",
               id="efdde334-fe7b-11e4-a322-1697f925ec7b",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               headers="",
               signature="MRlPr/Z1WQY2sMthcaEqETRMw4gPYXlPcTpaLWS2gcc="
```

Other headers = 
```
X-Acquia-Timestamp: 1432075982
```

Signature-Base-String =
```
GET
example.acquiapipet.net
/v1.0/task-status/133
limit=10
id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&version=2.0
1432075982
```

note that content type and body hash are omitted for GET.

#### POST Example

https://example.acquiapipet.net/v1.0/task/

Other headers:
```
X-Acquia-Timestamp: 1432075982
Content-Type: application/json
X-Acquia-Content-SHA256: 6paRNxUA7WawFxJpRp4cEixDjHq3jfIKX072k9slalo=
```

body:
```
{"method":"hi.bob","params":["5","4","8"]}
```

Authorization header =
```
Authorization: acquia-http-hmac realm="Pipet%20service",
               id="efdde334-fe7b-11e4-a322-1697f925ec7b",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               signature="df5m8PBJj5porD3Tkg8nxcQnNMA5wj9H5btygdRnABE="
```

Signature-Base-String =
```
POST
example.acquiapipet.net
/v1.0/task

id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&version=2.0
1432075982
application/json
9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI=
```

## Response Validation

Except for HEAD requests, the reponse from the server must include the following header

Response header =
```
X-Acquia-Content-HMAC-SHA256: UPiRBF/yd6po9Sv+1tBH5QmofBhQfm1R33okf4VyZtg=
```

The client should verify the reponse HMAC which authenticates the response body back from the server.

#### Response Signature Base String


The signature base string is a concatenated string generated from the following parts:

* `Nonce`:  The nonce that was sent in the Authorization header.
* `Body`: The response body (or empty string).

Signature-Base-String =

```
d1954337-5319-4821-8427-115542e08d10
{"id": 133, "status": "done"}
```


## Further Reading

Refer to the [wiki](https://github.com/acquia/http-hmac-spec/wiki)
for frequently asked questions and a list of implementations in various languages.

## Attribution

The algorithm is modeled after [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search) and [OAuth 1.0a](http://oauth.net/core/1.0a/) [RFC 5849](http://tools.ietf.org/html/rfc5849).
