# HTTP HMAC Spec

This spec documents an HMAC message format for securing RESTful web APIs.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

## Spec

Services implementing this spec must only accept requests using HTTPS.

## Overview of Header and Signature

The pseudocode below illustrates construction of the HTTP "Authorization" header and signature:

```
Authorization: acquia-http-hmac realm="Example",
               id="identifier",
               timestamp="1432075982.782971",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version=2.0,
               signature="Signature"

X-Acquia-Content-SHA256:

Signature = Base64( HMAC( SecretKey, Signature-Base-String ) );

Signature-Base-String =
    HTTP-Verb + "\n" +
    host  + "\n" +
    Path + "\n" +
    Header-Parameters + "\n" +
    GET-Parameters  OR
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
* `timestamp`: A Unix timestamp (which may be treated by the server as part of the nonce) represented as a float with 6 digit (microtime) precision
* `nonce`:  a hex version 4 (or version 1) UUID.
* `version`: the version of this spec
* `signature`: the Signature (base64 encoded) as described below.

Each value should be enclosed in double quotes and urlencoded (percent encoded).

Note that the name of this (standard) header is misleading - it carries authentication information.

#### X-Acquia-Content-SHA256 Header

The SHA-256 hash value used to generate the signature base string. This is analogous to the standard Content-MD5 header.

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
* `Header-Parameters`: normalized parameters similar to section 9.1.1 of OAuth 1.0a.  The parameters are the realm, version, id, and timestamp from the Authorization header. Parameters are sorted by name and separated by '&' with name and value separated by =, percent encoded
* `Parameters`: normalized parameters similar to section 9.1.1 of OAuth 1.0a.  Any GET query parameters.  Parameters are sorted by name and separated by '&' with name and value separated by =, percent encoded.
* `Content-Type`: The lowercase value of the "Content-type" header, or omit for a GET or HEAD request.
* `Body-Hash`: SHA-256 digest of the raw body of the HTTP request, for POST, PUT, PATH or other requests with a body, or an empty string for GET or HEAD.

#### GET Example

https://example.acquiapipet.net/v1.0/task-status/133?limit=10

Assuming the client ID is efdde334-fe7b-11e4-a322-1697f925ec7b and secret key is W5PeGMxSItNerkNFqQMfYiJvH14WzVJMy54CPoTAYoI=

Authorization header =
```
Authorization: acquia-http-hmac realm="Pipet%20service",
               id="efdde334-fe7b-11e4-a322-1697f925ec7b",
               timestamp="1432075982.765341",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               signature="9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI="
```

Signature-Base-String =
```
GET
example.acquiapipet.net
/v1.0/task-status/133
id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&timestamp=1432075982.782971&version=2.0
limit=10
```

note that content type and body hash are omitted for GET.

#### POST Example

https://example.acquiapipet.net/v1.0/task/

non-auth headers:
```
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
               timestamp="1432075982.765341",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               signature="9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI="
```

Signature-Base-String =
```
POST
example.acquiapipet.net
/v1.0/task
id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&timestamp=1432075982.782971&version=2.0
application/json
9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI=
```


## Further Reading

Refer to the [wiki](https://github.com/acquia/http-hmac-spec/wiki)
for frequently asked questions and a list of implementations in various languages.

## Attribution

The algorithm is modeled after [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search) and [OAuth 1.0a](http://oauth.net/core/1.0a/) [RFC 5849](http://tools.ietf.org/html/rfc5849).
