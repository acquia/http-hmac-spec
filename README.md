# HTTP HMAC Spec

This spec documents an HMAC message format for securing RESTful web APIs.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

## Spec

The pseudocode below illustrates construction of the HTTP "Authorization" header and signature:

```
Authorization = "Provider" + " " + ID + ":" + Signature;

Signature = Base64( Algorithm( SecretKey, Message ) );

Message =
    HTTP-Verb + "\n" +
    MD5( Request-Body ) + "\n" +
    Content-Type + "\n" +
    Date + "\n" +
    Custom-Headers + "\n"
    Resource
;
```

`"\n"` denotes a Unix-style line feed (ASCII code `0x0A`).

#### Authorization Header

The value of the `Authorization` header contains the following parts:

* `Provider`: The provider, for example "Acquia", "MyCompany", etc.
* `ID`: The API key's unique identifier, which is an arbitrary string
* `Signature`: The base64 encoded HMAC digest as described below

#### Signature

The signature is a base64 encoded binary HMAC digest generated from the
following parts:

* `Algorithm`: The cryptographic algorithm, e.g. "sha1", "sha256", etc.
* `SecretKey`: The API key's shared secret
* `Message`: The string being signed as described below

#### Message

The message is a concatenated string  generated from the following parts:

* `HTTP-Verb`: The uppercase HTTP request method e.g. "GET", "POST"
* `Request-Body`: The raw body of the HTTP request
  * Note that in the message, the body is hashed using the MD5 algorithm
* `Content-Type`: The lowercase value of the "Content-type" header
* `Date`: The value of the "Date" header
  * The implementation may also read the timestamp from a custom header, e.g. `x-acquia-timestamp`
* `Custom-Headers`: A canonicalized concatenation of optional custom headers
  * Each header is in `x-custom-header: value` format
  * Header names are lowercase
  * Multiple values are separated by a comma and space, e.g. `value1, value2`
  * Each header is separated by a newline in the concatenated string
* `Resource`: The HTTP request path + query string, e.g. `/resource?key=value`

## Further Reading

Refer to the [wiki](https://github.com/acquia/http-hmac-spec/wiki)
for frequently asked questions and a list of implementations in various languages.

## Attribution

The algorithm is modeled after [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search).
