# HTTP HMAC Spec

This spec documents an HMAC message format for securing RESTful web APIs.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

## Spec

The following pseudocode illustrates the construction of the HTTP
Authorization header and signature.

```
Authorization = "Provider" + " " + ID + ":" + Signature;

Signature = Base64( HMAC-SHA1 )( SecretKey, Message ) );

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

* `SecretKey`: The API key's shared secret
* `Message`: The string being signed as described below

#### Message

The message is a concatenated string  generated from the following parts:

* `HTTP-Verb`: The uppercase HTTP request method e.g. "GET", "POST"
* `Request-Body`: The raw body of the HTTP request
* `Content-Type`: The lowercase value of the "Content-type" header
  * Note that in the message, the body is hashed using the MD5 algorithm
* `Date`: The value of the "Date" header
  * The implementation may also read the timestamp from a custom header, e.g. `x-acquia-timestamp`
* `Custom-Headers`: A canonicalized concatenation of specified custom headers
  * Each header is in `x-custom-header: value` format
  * Header names are lowercase
  * Headers with multiple multiple values are separated by a comma and space, e.g. `value1, value2`
  * Each header is separated by a newline in the concatenated string
* `Resource`: The HTTP request path + query string, e.g. `/resource?key=value`

## FAQ

#### Why not HTTP basic authentication?

Basic authentication is the simplest way to add authentication to a REST API,
however it is generally considered the least secure authentication method since
the same hashed password is sent on every API request.

#### Why not OAuth 1.0a?

OAuth 1.0a is a widely adopted protocol that also uses an HMAC-based algorithm
to sign and authenticate API requests. The main security advantage that OAuth
1.0a has over bare HMAC authentication systems is the "request token" workflow
that enables browsers to initiate authentication requests on behalf of a server
without ever being passed the shared secret.

The downside of this technique is the overall complexity it adds by requiring
the application making requests to implement the OAuth 1.0a protocol as well. If
passing the shared secret in a browser is not a concern for the app, then bare
HMAC authentication systems can provide equivalent security with less
complexity.

#### Why not OAuth 2.0?

This is best explained by Eran Hammer's [OAuth 2.0 and the Road to Hell](http://hueniverse.com/2012/07/26/oauth-2-0-and-the-road-to-hell/)
blog post explaining why he resigned as lead author and editor of the spec.

## Attribution

The algorithm is modeled after [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search).
