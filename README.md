Note: This is inspired on the Hawk Spec and documentation.

# HTTP HMAC Spec

This spec documents an HMAC authentication format for securing RESTful web APIs.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

Current version: **2.0**

Note: 2.0 is not backwards compatible to 1.0 as the message to sign and the Headers required are wildy different.

# Table of Content

- [**Introduction**](#introduction)
- [**Specification**](#spec)
- [**Overview of Request and Response Headers**](#overview-of-request-and-response-headers)
  - [Client](#client)
  - [Server](#server)
- [**Overview of Request Header and Signature**](#overview-of-request-header-and-signature)
- [**Overview of Response Header and Signature**](#overview-of-response-header-and-signature)
- [**Examples**](#examples)
  - [Request Examples](#request-examples)
  - [Response Examples](#response-examples)
- [**Security Considerations**](#security-considerations)
  - [MAC Keys Transmission](#mac-keys-transmission)
  - [Confidentiality of Requests](#confidentiality-of-requests)
  - [Spoofing by Counterfeit Servers](#spoofing-by-counterfeit-servers)
  - [Plaintext Storage of Credentials](#plaintext-storage-of-credentials)
  - [Entropy of Keys](#entropy-of-keys)
  - [Coverage Limitations](#coverage-limitations)
  - [Future Time Manipulation](#future-time-manipulation)
  - [Client Clock Poisoning](#client-clock-poisoning)
  - [Host Header Forgery](#host-header-forgery)
<p></p>
- [**Frequently Asked Questions**](#frequently-asked-questions)
<p></p>
- [**Further Reading**](#further-reading)
- [**Implementations**](#implementations)
- [**Testing**](#testing)
- [**Attribution**](#attribution)

## Introduction
TBD, see the Hawk spec for inspiration

## Spec

Production services implementing this spec must only accept requests using HTTPS.

## Overview of Request and Response Headers

### Client
The HTTP client signs the request by adding the following HTTP headers:

```
Authorization : <HMACAuthorization>
X-Authorization-Timestamp : <Unix Timestamp In Seconds>
X-Authorization-Content-SHA256 : <HashedContent> (if Content-Length > 0)
```
The HTTP client must not add the following HTTP Header to the request:

X-Authenticated-Id (reserved for internal server to server communication)

### Server

If the server authenticates the request successfully, it will add the following HTTP headers to the response (for all non-HEAD requests):

```
X-Server-Authorization-HMAC-SHA256: <HMACServerAuthorization>
```

The client should verify the response HMAC which authenticates the response body back from the server.

## Overview of Request Header and Signature

The pseudocode below illustrates construction of the HTTP "Authorization" header and signature:

```
HMACAuthorization = "acquia-http-hmac" + " " +
                "realm=" + DoubleQuoteEnclose( URLEncode( Provider ) ) + "," +
                "id=" + DoubleQuoteEnclose( URLEncode( AccessKey ) ) + "," +
                "nonce=" + DoubleQuoteEnclose( URLEncode( HexV4OfRandomUUID ) )+ "," +
                "version=" + DoubleQuoteEnclose( URLEncode( HMACVersion ) ) + "," +
                "headers=" + DoubleQuoteEnclose( URLEncode( AdditionalSignedHeaderNames ) ) + "," +
                "signature=" + DoubleQuoteEnclose( URLEncode( HMACSignature ) ) 

AdditionalSignedHeaderNames = "" or
    Lowercase( HTTP-Header-Name ) [ + ";" + Lowercase( HTTP-Header-Name ), for additional headers]
                
HMACSignature = Base64( HMAC-SHA256 ( SecretKey, UTF-8-Encoding-Of( StringToSign ) ) )

StringToSign = HTTP-Verb + "\n" +
   Host + "\n" +
   Path + "\n" +
   QueryParameters + "\n" +
   AuthorizationHeaderParameters + "\n" +
   [ AdditionalSignedHeaders, If the headers attribute is not empty, + "\n" ] + 
   X-Authorization-Timestamp +
   [ "\n" + Content-Type + 
     "\n" + HashedContent, if Content-Length > 0 ]

AuthorizationHeaderParameters = "id=" + URLEncode( AccessKey ) + "&" + 
   "nonce=" + URLEncode( HexV4OfRandomUUID ) + "&" +
   "realm=" + URLEncode( Provider ) + "&" +
   "version=" + URLEncode( Version )
   
AdditionalSignedHeaders = Lowercase( HTTP-Header-Name ) + ":" + HTTP-Header-Value 
   [ + "\n" + Lowercase( HTTP-Header-Name ) + ":" + HTTP-Header-Value, for additional headers]
    (must be sorted by Lowercase( HTTP-Header-Name ) )

HashedContent = Base64( SHA256 ( Request-Body ) )
```

`"\n"` denotes a Unix-style line feed (ASCII code `0x0A`).

### Authorization Header

The value of the `Authorization` header contains the following attributes:

* `realm`: The provider, for example "Acquia", "MyCompany", etc.
* `id`: The API key's unique identifier, which is an arbitrary string
* `nonce`:  a hex version 4 (or version 1) [UUID](http://tools.ietf.org/rfc/rfc4122.txt).
* `version`: the version of this spec
* `headers`: a list of additional request headers that are to be included in the signature base string. These are lower-case, and separated with ;
* `signature`: the Signature (base64 encoded) as described below.

Each attribute value should be enclosed in double quotes and urlencoded (percent encoded).

Note that the name of this (standard) header is misleading - it carries authentication information.

#### Signature

The signature is a base64 encoded binary HMAC-SHA256 digest generated from the
following parts:

* `SecretKey`: The API key's shared secret
* `StringToSign`: The string being signed as described below

#### Secret Key

The secret key should be a 256 to 512 bit binary value. The secret key will normally be stored as a base64-encoded or hex-encoded string representation, but must be decoded to the binary value before use.

#### String To Sign

The signature base string is a concatenated string generated from the following parts:

* `HTTP-Verb`: The uppercase HTTP request method e.g. "GET", "POST"
* `Host`: The (lowercase) hostname, matching the HTTP "Host" request header field (including any port number)
* `Path`: The HTTP request path with leading slash, e.g. `/resource/11`
* `QueryParameters`: Any query parameters or empty string. This should be the exact string sent by the client, including urlencoding.
* `AuthorizationHeaderParameters`: normalized parameters similar to section 9.1.1 of OAuth 1.0a.  The parameters are the id, nonce, realm, and version from the Authorization header. Parameters are sorted by name and separated by '&' with name and value separated by =, percent encoded (urlencoded)
* `AdditionalSignedHeaders`: The normalized header names and values specified in the headers parameter of the Authorization header. Names should be lower-cased, sorted by name, separated from value by a colon and the value followed by a newline so each extra header is on its own line. If there are no added signed headers, an empty line should not be added to the signature base string.
* `X-Authorization-Timestamp`:  The value of the X-Authorization-Timestamp header
* `Content-Type`: The lowercase value of the "Content-type" header (or empty string if absent). Omit if Content-Length is 0.
* `HashedContent`: The base64 encoded SHA-256 digest of the raw body of the HTTP request, for POST, PUT, PATCH, DELETE or other requests that may have a body. Omit if Content-Length is 0. This should be identical to the string sent as the X-Authorization-Content-SHA256 header.
  
#### X-Authorization-Timestamp Header

A Unix timestamp (integer seconds since Jan 1, 1970 UTC). Required for all requests. If this value differs by more than 900 seconds (15 minutes) from the time of the server, the request will be rejected.

#### X-Authorization-Content-SHA256 Header

The base64 encoded SHA-256 hash value used to generate the signature base string. This is analogous to the standard Content-MD5 header. Required for any request where Content-Length is not 0 (for example, a POST request with a body).

#### X-Authenticated-Id Header

If the X-Authenticated-Id is present in the request, the client implementing the validation of the request should reject the request and return "unauthenticated". This header is reserved for servers or proxies who want to validate requests and forward requests to backends. Backends can read this added header to understand if it was authenticated. Use this with caution and careful consideration as adding this header only guarantees it was authenticated to that ID.

## Overview of Response Header and Signature

The pseudocode below illustrates construction of the HTTP "X-Server-Authorization-HMAC-SHA256" header and signature for all non-HEAD requests:

```
HMACServerAuthorization = Base64( HMAC-SHA256 ( SecretKey, UTF-8-Encoding-Of( ResponseStringToSign ) ) )

ResponseStringToSign = Nonce + "\n" +
    X-Authorization-Timestamp + "\n" +
    Response-Body 
```

#### HMAC Server Authorization

The server authorization is a base64 encoded binary HMAC-SHA256 digest generated from the
following parts:

* `SecretKey`: The API key's shared secret
* `ResponseStringToSign`: The string being signed as described below

### Response String To Sign

The response signature base string is a concatenated string generated from the following parts:

* `Nonce`:  The nonce that was sent in the Authorization header.
* `X-Authorization-Timestamp`: The timestamp that was sent in the X-Authorization-Timestamp header
* `Response-Body`: The response body (or empty string).

## Examples

### Request Examples 

#### GET Example

Make a GET request to https://example.acquiapipet.net/v1.0/task-status/133?limit=10 with the id = 'efdde334-fe7b-11e4-a322-1697f925ec7b' and Secret Key (Base64 Encoded) = 'W5PeGMxSItNerkNFqQMfYiJvH14WzVJMy54CPoTAYoI=' on Thursday 19 May 2015 22:53:02 GMT (Unix Timestamp = 1432075982) with the realm = 'Pipet service'

The following X-Authorization-Timestamp header will be added to the HTTP request
```
X-Authorization-Timestamp: 1432075982
```

The following StringToSign, with a randomly generated nonce (in this example nonce = d1954337-5319-4821-8427-115542e08d10), will be built
```
GET
example.acquiapipet.net
/v1.0/task-status/133
limit=10
id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&version=2.0
1432075982
```

The following Authorization header will be added to the HTTP request (the signature is based on the StringToSign and SecretKey)
```
Authorization: acquia-http-hmac realm="Pipet%20service",
               id="efdde334-fe7b-11e4-a322-1697f925ec7b",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               headers="",
               signature="MRlPr/Z1WQY2sMthcaEqETRMw4gPYXlPcTpaLWS2gcc="
```

#### POST Example


Make a POST request to https://example.acquiapipet.net/v1.0/task/ with the id = 'efdde334-fe7b-11e4-a322-1697f925ec7b' and Secret Key (Base64 Encoded) = 'W5PeGMxSItNerkNFqQMfYiJvH14WzVJMy54CPoTAYoI=' on Thursday 19 May 2015 22:53:02 GMT (Unix Timestamp = 1432075982) with the realm = 'Pipet service' and the following HTTP headers/body:

Request Headers
```
Content-Type: application/json
```

Request Body
```
{"method":"hi.bob","params":["5","4","8"]}
```

The following X-Authorization-Timestamp header will be added to the HTTP request
```
X-Authorization-Timestamp: 1432075982
```

The following X-Authorization-Content-SHA256 header will be calculated from the Request Body and added to the HTTP request
```
X-Authorization-Content-SHA256: 9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI=
```

The following StringToSign, with a randomly generated nonce (in this example nonce = d1954337-5319-4821-8427-115542e08d10), will be built
```
POST
example.acquiapipet.net
/v1.0/task

id=efdde334-fe7b-11e4-a322-1697f925ec7b&nonce=d1954337-5319-4821-8427-115542e08d10&realm=Pipet%20service&version=2.0
1432075982
application/json
9tn9ZdUBc0BgXg2UdnUX7bi4oTUL9wakvzwBN16H+TI=
```

The following Authorization header will be added to the HTTP request (the signature is based on the StringToSign and Secret Key)
```
Authorization: acquia-http-hmac realm="Pipet%20service",
               id="efdde334-fe7b-11e4-a322-1697f925ec7b",
               nonce="d1954337-5319-4821-8427-115542e08d10",
               version="2.0",
               signature="XDBaXgWFCY3aAgQvXyGXMbw9Vds2WPKJe2yP+1eXQgM="
```

### Response Examples

#### Server Response Example

Return a response based on a non-HEAD request made on Thursday 19 May 2015 22:53:02 GMT (Unix Timestamp = 1432075982) with the request's nonce = 'd1954337-5319-4821-8427-115542e08d10', Secret Key (Base64 Encoded) = 'W5PeGMxSItNerkNFqQMfYiJvH14WzVJMy54CPoTAYoI=', and the following HTTP response body:

Response Body
```
{"id": 133, "status": "done"}
```

The following ResponseStringToSign will be built
```
d1954337-5319-4821-8427-115542e08d10
1432075982
{"id": 133, "status": "done"}
```

The following X-Server-Authorization-HMAC-SH256 will be added to HTTP response (the value is based on the ResponseStringToSign and Secret Key)
```
X-Server-Authorization-HMAC-SHA256: M4wYp1MKvDpQtVOnN7LVt9L8or4pKyVLhfUFVJxHemU=
```


## Security Considerations

The greatest sources of security risks are usually found not in the Auth Protocol but in the policies and procedures surrounding its use.
Implementers are strongly encouraged to assess how this module addresses their security requirements. This section includes
an incomplete list of security considerations that must be reviewed and understood before deploying this Auth protocol on the server.
Many of the protections provided in the specifications depends on whether and how they are used.

### MAC Keys Transmission

**Http Hmac Spec** does not provide any mechanism for obtaining or transmitting the set of shared credentials required. Any mechanism used to obtain **Http Hmac Spec** credentials must ensure that these transmissions are protected using transport-layer mechanisms such as TLS.

### Confidentiality of Requests

While **Http Hmac Spec**provides a mechanism for verifying the integrity of HTTP requests, it provides no guarantee of request confidentiality. Unless other precautions are taken, eavesdroppers will have full access to the request content. Servers should carefully consider the types of data likely to be sent as part of such requests, and employ transport-layer security mechanisms to protect sensitive resources.

### Spoofing by Counterfeit Servers

**Http Hmac Spec** provides limited verification of the server authenticity. When receiving a response back from the server, the server may choose to include a response `X-Server-Authorization-HMAC-SHA256` header which the client can use to verify the response. However, it is up to the server to determine when such measure is included, to up to the client to enforce that policy.

A hostile party could take advantage of this by intercepting the client's requests and returning misleading or otherwise
incorrect responses. Service providers should consider such attacks when developing services using this protocol, and should
require transport-layer security for any requests where the authenticity of the resource server or of server responses is an issue.

### Plaintext Storage of Credentials

The **Http Hmac Spec** key functions the same way passwords do in traditional authentication systems. In order to compute the request MAC, the server must have access to the key in plaintext form. This is in contrast, for example, to modern operating systems, which store only a one-way hash of user credentials.

If an attacker were to gain access to these keys - or worse, to the server's database of all such keys - he or she would be able to perform any action on behalf of any resource owner. Accordingly, it is critical that servers protect these keys from unauthorized access.

### Entropy of Keys

Unless a transport-layer security protocol is used, eavesdroppers will have full access to authenticated requests and request
MAC values, and will thus be able to mount offline brute-force attacks to recover the key used. Servers should be careful to
assign keys which are long enough, and random enough, to resist such attacks for at least the length of time that the **Http Hmac Spec** credentials are valid.

For example, if the credentials are valid for two weeks, servers should ensure that it is not possible to mount a brute force
attack that recovers the key in less than two weeks. Of course, servers are urged to err on the side of caution, and use the
longest key reasonable.

It is equally important that the pseudo-random number generator (PRNG) used to generate these keys be of sufficiently high
quality. Many PRNG implementations generate number sequences that may appear to be random, but which nevertheless exhibit
patterns or other weaknesses which make cryptanalysis or brute force attacks easier. Implementers should be careful to use
cryptographically secure PRNGs to avoid these problems.

### Coverage Limitations

The request MAC only covers the HTTP `Host` header, the `Content-Type` header and optionally a given set of Headers. It does not cover any other headers that it does not know about which can often affect how the request body is interpreted by the server. If the server behavior is influenced by the presence or value of such headers, an attacker can manipulate the request headers without being detected. Implementers should use the `headers` feature to pass headers to be added to the signature via the `Authorization` header which is protected by the request MAC. The Signature Base String will then be responsible of adding these given headers to the Signature so they become part of the MAC.

The response authentication, when performed, only covers the response Body (payload) and some of the request information
provided by the client in it's request such as timestamp and nonce. It does not cover the HTTP status code or
any other response header field (e.g. Location) which can affect the client's behaviour.

### Future Time Manipulation

If an attacker is able to manipulate this information and cause the client to use an incorrect time, it would be able to cause the client to generate authenticated requests using time in the future. Such requests will fail when sent by the client, and will not likely leave a trace on the server (given the common implementation of nonce, if at all enforced). The attacker will then be able to replay the request at the correct time without detection.

A solution to this problem would be a clock sync between the client and server. To accomplish this, the server informs the client of its current time when an invalid timestamp is received. This happens in the form of a Date header in the response. See [RFC2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.18) as a motivation for this.

The client should only use the time information provided by the server if:
* it was delivered over a TLS connection and the server identity has been verified, or
* the MAC digest calculated using the same client credentials over the timestamp has been verified. (eg Response validation was successful)

### Client Clock Poisoning

When receiving a request with a bad timestamp, the server provides the client with its current time. The client must never use the time received from the server to adjust its own clock, and must only use it to calculate an offset for communicating with that particular server.

### Host Header Forgery

**Http Hmac Spec** validates the incoming request MAC against the incoming HTTP Host header. A malicous client can mint new host names pointing to the server's IP address and use that to craft an attack by sending a valid request that's meant for another hostname than the one used by the server. Server implementors must manually verify that the host header received matches their expectation. Eg, if you expect API calls on test.myapi.com, verify that that is the domain the request was sent to in the server implementation.

## Frequently Asked Questions

### Where is the protocol specification?

If you are looking for some prose explaining how all this works, **this is it**. **Http Hmac Spec** is being developed as an open source project instead of a standard. In other words, this document is the specification. Not sure about something? Open an issue!

### How is this different from [hawk](https://github.com/hueniverse/hawk)?
* Allows to add a list of headers to be added to the signature.
* Removes any support for bewit requests.

### Is it done?

No, it is still in development and we are trying to validate our assumptions every day.

### Why not SHA1 or other algorithms to make the HMAC?

SHA256 is a standard that is sufficient for message passing. It also doesn't require a lot of processing power to calculate the hash compared to more advanced protocols. We are confident that SHA256 is the best alternative we could choose and is also future proof. Therefor, we do not allow a custom algorithm in the request or in the protocol.

### Why is request param normalization not needed? OAuth 2.0 requires it.
Eran Hammer was so friendly to guide us to this answer. Request Parameters normalization was done for PHP4 support which did not provide access to the raw request URI or raw form encoded payload. Time has changed now and any language used should have direct access to the request params.

### Why not just use HTTP Digest?

Digest requires pre-negotiation to establish a nonce. This means you can't just make a request - you must first send
a protocol handshake to the server. This pattern has become unacceptable for most web services, especially mobile
where extra round-trip are costly.

### Why bother with all this nonce and timestamp business?

**Http Hmac Spec**, just like **Hawk** is an attempt to find a reasonable, practical compromise between security and usability. OAuth 1.0 got timestamp and nonces halfway right but failed when it came to scalability and consistent developer experience. **Http Hmac Spec** addresses it by requiring the client to sync its clock, but provides it with tools to accomplish it.

In general, replay protection is a matter of application-specific threat model. It is less of an issue on a TLS-protected
system where the clients are implemented using best practices and are under the control of the server. Instead of dropping
replay protection, **Http Hmac Spec** offers a required time window and an optional nonce verification. Together, it provides developers with the ability to decide how to enforce their security policy without impacting the client's implementation.

## Further Reading

Refer to the [wiki](https://github.com/acquia/http-hmac-spec/wiki)
for frequently asked questions and a list of implementations in various languages.

## Implementations
List Auth Implementations here (go, php, bash, etc..)

## Testing

This version of the spec can be tested using the [test fixtures file](fixtures.json). These fixtures provide the required inputs and expectations to ensure that the different implementations are compatible.

## Attribution

The algorithm is modeled after [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search). Elements of the spec were also informed by [OAuth 1.0a](http://oauth.net/core/1.0a/), [RFC 5849](http://tools.ietf.org/html/rfc5849), and [Hawk](https://github.com/hueniverse/hawk)
