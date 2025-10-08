---
title: "HTTP Message Signatures Directory"
abbrev: "HTTP Message Signatures Directory"
category: std

docname: draft-meunier-http-message-signatures-directory-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Web Bot Auth"
keyword:
 - not-yet
venue:
  group: "Web Bot Auth"
  type: "Working Group"
  mail: "web-bot-auth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-http-message-signatures-directory.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  HTTP: RFC9110
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP-MESSAGE-SIGNATURES-IANA:
    title: HTTP Message Signatures
    target: https://www.iana.org/assignments/http-message-signature/http-message-signature.xhtml
  JWK: RFC7517
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638
  STRUCTURED-HEADERS: RFC8941
  URI: RFC8820
  WellKnownURIs:
    title: Well-Known URIs
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml

informative:
  BASE64: RFC2397
  CRYPTO-TEST-KEYS: RFC9500
  X509-PKI: RFC5280


--- abstract

This document describes a method for clients using {{HTTP-MESSAGE-SIGNATURES}}
to advertise their signing keys.

It defines a key directory format based on JWKS as defined in {{Section 5 of JWK}},
as well as new HTTP Method Context to allow for in-band key discovery.


--- middle

# Introduction

{{HTTP-MESSAGE-SIGNATURES}} allow a signer to generate a signature over an HTTP message, and a verifier to validate it.
The specification assumes verifiers have prior knowledge of
signers' key material, requiring out-of-band key distribution mechanisms. This creates deployment
friction and limits the ability to dynamically verify signatures from previously unknown signers.

This document defines:

1. A standardized key directory format based on JWKS for publishing HTTP Message Signatures keys,
2. A well-known URI location for discovering these key directories,
3. A new HTTP header field enabling in-band key directory location discovery.

Together, these mechanisms enable key distribution and discovery for HTTP Message Signatures cryptographic material.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Configuration {#configuration}

The key directory is served as a JSON Web Key Set (JWKS) as defined in {{Section 5 of JWK}}.
The "alg" parameter are restricted to algorithm registered against HTTP Signature Algorithms Section of {{HTTP-MESSAGE-SIGNATURES-IANA}}

The directory SHOULD be served over HTTPS.
The directory MUST be served with media type `application/http-message-signatures-directory+json`.

A client application SHOULD validate the directory format and reject malformed entries.

# HTTP Method Context `Signature-Agent`

A service sending signed requests as defined in {{HTTP-MESSAGE-SIGNATURES}} MAY include a
`Signature-Agent` header field to communicate its signing key directory. This header
field contains a URI allowing retrieval of an HTTP Message Signatures Directory as
defined in {{configuration}}.

## Header Field Definition

The `Signature-Agent` header field is an Dictionary Structured Header as defined
in {{Section 3.2 of STRUCTURED-HEADERS}}.
Its member values MUST be an String Item which contain a {{URI}}.

The URI scheme MUST be one of:

- **https (RECOMMENDED)**: Points to an HTTPS resource serving the key directory
- **http**: Points to an HTTP resource serving the key directory
- **data**: Contains an inline key directory

When using the "data" URI scheme, the media type MUST be
`application/http-message-signatures-directory+json`. The content MAY be base64 encoded
as per {{BASE64}}.

If dictonary values are not a valid URI-reference, the entire header field MAY be
ignored.

# Security Considerations {#security}

## Key rotation

Clients SHOULD implement key rotation by including multiple keys in the directory
with a different validity period. When rotating keys, clients SHOULD:

1. Add the new key to the directory before its intended use date
2. Continue to include the old key until its expiration date
3. Remove expired keys from the directory

Servers SHOULD cache the directory contents and refresh upon expiration.

## Binding keys to the directory authority

To ensure the authenticity and integrity of the key material provided by the
directory, clients **SHOULD** validate the directory's response.

When a directory server provides a key directory over HTTP or HTTPS, it is
RECOMMENDED that it constructs and includes one HTTP Message Signatures per keys
with the response, as defined in {{HTTP-MESSAGE-SIGNATURES}}.
Each key SHOULD be used to provide one signature.

Directory server SHOULD include:

`@authority`
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}

Directory server SHOULD include the following `@signature-params` as defined in
{{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`keyid`
: MUST be a base64url JWK SHA-256 Thumbprint as defined in {{Section 3.2 of JWK-THUMBPRINT}} for RSA and EC, and in {{Appendix A.3 of JWK-OKP}} for ed25519.

`tag`
: MUST be `http-message-signatures-directory`

Clients SHOULD validate these signatures using the keys provided by the
directory. Clients SHOULD ignore keys from a directory response that do not have
a corresponding valid signature. This validation ensures the integrity of the
key set and its association with the intended directory.

# Privacy Considerations

Key directories enable discovery of signing keys which may reveal information about the
signing entity. Implementers should consider:

## Directory Content
Key directories should only contain keys actively used for signing. Including additional
keys or metadata may expose unnecessary information about the signing service.

## Access Patterns
Verifiers accessing key directories may reveal information about signature verification
patterns. Directory servers should avoid logging personally identifiable information
from directory requests.


# IANA Considerations

This section contains considerations for IANA.

## Well-Known 'http-message-signatures-directory' URI {#wkuri-reg}

This document updates the "Well-Known URIs" Registry {{WellKnownURIs}} with the
following values.

| URI Suffix  | Change Controller  | Reference | Status | Related information |
|:------------|:-------------------|:----------|:-------|:--------------------|
| http-message-signatures-directory | IETF | [this document] | permanent | None |
{: #wellknownuri-values title="'http-message-signatures-directory' Well-Known URI"}

## Media Types

The following entries should be added to the IANA "media types"
registry:

- "application/http-message-signatures-directory+json"

The templates for these entries are listed below and the
reference should be this RFC.

### "application/http-message-signatures-directory+json" media type

Type name:

: application

Subtype name:

: http-message-signatures-directory

Required parameters:

: N/A

Optional parameters:

: N/A

Encoding considerations:

: "binary"

Security considerations:

: see {{security}}

Interoperability considerations:

: N/A

Published specification:

: this specification

Applications that use this media type:

: Services that implement the signer role for HTTP Message
  Signatures and verifiers that interact with the signer for
  the purpose of validating signatures.


Fragment identifier considerations:

: N/A

Additional information:

: <dl spacing="compact">
  <dt>Magic number(s):</dt><dd>N/A</dd>
  <dt>Deprecated alias names for this type:</dt><dd>N/A</dd>
  <dt>File extension(s):</dt><dd>N/A</dd>
  <dt>Macintosh file type code(s):</dt><dd>N/A</dd>
  </dl>

Person and email address to contact for further information:

: see Authors' Addresses section

Intended usage:

: COMMON

Restrictions on usage:

: N/A

Author:

: see Authors' Addresses section

Change controller:

: IETF
{: spacing="compact"}


--- back

# Examples

## Key Directory on example.com

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: example.com
Accept: application/http-message-signatures-directory+json

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory+json
Cache-Control: max-age=86400
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  }]
}
~~~

## Delegation and chaining

There are multiple methods to perform delegation and chaining. There are no specific methods
that have been favored by implementation so far, should they even support them.
It is adviced to consider delegation as experimental for now, and provide input on the associated
[GitHub issue](https://github.com/thibmeu/http-message-signatures-directory/issues/27).

### Key Directory on sub.example.com with a delegation from example.com via x5c full certificate chain

In this example, example.com key is testECCP256 provided in {{Section 2.3 of CRYPTO-TEST-KEYS}}.
Certificate chain is passed via x5c key parameter defined in {{Section 4.7 of JWK}}.

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: sub.example.com
Accept: application/http-message-signatures-directory

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory
Cache-Control: max-age=86400
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600,
    "x5c": [
      "MIIBYTCCAQagAwIBAgIUFDXRG3pgZ6txehQO2LT4aCqI3f0wCgYIKoZIzj0EAwIwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wHhcNMjUwNjEzMTA0MjQxWhcNMzUwNjExMTA0MjQxWjAaMRgwFgYDVQQDDA9zdWIuZXhhbXBsZS5jb20wKjAFBgMrZXADIQAmtAuPk//z2JcRL368WCsjLb1yUX0IL+g8+zDdzkPRu6NdMFswCQYDVR0TBAIwADAOBgNVHQ8BAf8EBAMCB4AwHQYDVR0OBBYEFKV3qaYNFbzQB1QmN4sa13+t4RmoMB8GA1UdIwQYMBaAFFtwp5gX95/2N9L349xEbCEJ17vUMAoGCCqGSM49BAMCA0kAMEYCIQC8r+GvvNnjI+zzOEDMOM/g9e8QLm00IZXP+tjDqah1UQIhAJHffLke9iEP1pUdm+oRLrq6bUqyLELi5TH2t+BaagKv",
      "MIIBcDCCARagAwIBAgIUS502rlCXxG2vviltGdfe3fmX4pIwCgYIKoZIzj0EAwIwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wHhcNMjUwNjEzMTA0MTQzWhcNMzUwNjExMTA0MTQzWjAWMRQwEgYDVQQDDAtleGFtcGxlLmNvbTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABEIlSPiPt4L/teyjdERSxyoeVY+9b3O+XkjpMjLMRcWxbEzRDEy41bihcTnpSILImSVymTQl9BQZq36QpCpJQnKjQjBAMA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgIEMB0GA1UdDgQWBBRbcKeYF/ef9jfS9+PcRGwhCde71DAKBggqhkjOPQQDAgNIADBFAiEAwTOqm1zNAvZuQ8Zb5AftQIZotq4Xe6GHz3+nJ04ybgoCIEEZtn1Pa+GCbmbWh12piHJBKh09TCA0feTedisbwzPV"
    ]
  }]
}
~~~

### Key Directory on sub.example.com with a delegation from example.com via a leaf certificate and AIA field

In this example, example.com key is testECCP256 provided in {{Section 2.3 of CRYPTO-TEST-KEYS}}.
Certificate chain is passed via x5c key parameter defined in {{Section 4.7 of JWK}},
and the root certificate is signaled by the presence of an Authority Information Access extension
as defined in {{Section 5.2.7 of X509-PKI}}.

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: sub.example.com
Accept: application/http-message-signatures-directory

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory
Cache-Control: max-age=86400
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600,
    "x5c": [
      "MIIBYTCCAQagAwIBAgIUFDXRG3pgZ6txehQO2LT4aCqI3f0wCgYIKoZIzj0EAwIwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wHhcNMjUwNjEzMTA0MjQxWhcNMzUwNjExMTA0MjQxWjAaMRgwFgYDVQQDDA9zdWIuZXhhbXBsZS5jb20wKjAFBgMrZXADIQAmtAuPk//z2JcRL368WCsjLb1yUX0IL+g8+zDdzkPRu6NdMFswCQYDVR0TBAIwADAOBgNVHQ8BAf8EBAMCB4AwHQYDVR0OBBYEFKV3qaYNFbzQB1QmN4sa13+t4RmoMB8GA1UdIwQYMBaAFFtwp5gX95/2N9L349xEbCEJ17vUMAoGCCqGSM49BAMCA0kAMEYCIQC8r+GvvNnjI+zzOEDMOM/g9e8QLm00IZXP+tjDqah1UQIhAJHffLke9iEP1pUdm+oRLrq6bUqyLELi5TH2t+BaagKv"
    ]
  }]
}
~~~

The AIA extension is as follow

~~~
X509v3 extensions:
  Authority Information Access:
    CA Issuers - URI:https://example.com/.well-known/http-message-signatures-directory.crt
~~~

<!-- Words from @sandormajor. TODO: add acknowledgement -->

The verifier should validate the signature with the public key in the Signature-Agent,
match the public key with the leaf cert, then fetch the root cert from the AIA URI and verify the leaf cert with it.

### Key Directory on sub.example.com with a delegation from example.com via x5u field

Leveraging x5c imposes that a PEM encoded certificate is present in the returned JWKS.
If size is a constraint, or deployment imposes a more dynamic certificate management,
directory server may use x5u key parameter defined in {{Section 4.6 of JWK}}.

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: sub.example.com
Accept: application/http-message-signatures-directory

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory
Cache-Control: max-age=86400

{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600,
    "x5u": "https://example.com/.well-known/http-message-signature-chain/sub.example.com.crt"
}
~~~

## Request with HTTP Signature-Agent

This extend the examples from {{Appendix B of HTTP-MESSAGE-SIGNATURES}}.

~~~
POST /foo?param=Value&Pet=dog HTTP/1.1
Host: example.com
Signature-Agent: my_test="https://directory.test"
{"hello": "world"}

HTTP/1.1 200 OK
{"message": "good dog"}
~~~

## Request with `data` URI Signature-Agent

A Signature-Agent using `data` URI can be used to communicate an ephemeral keys, as long as there is a chain to a certificate trusted by the origin.

In this example, the directory is signed by `example.com`. The CA is self-signed, even though it MAY be part of an existing PKI.

~~~
POST /foo?param=Value&Pet=dog HTTP/1.1
Host: example.com
Signature-Agent: my_test="data:application/http-message-signatures-directory;utf8,{\"keys\":[{\"kty\":\"OKP\",\"crv\":\"Ed25519\",\"kid\":\"NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw\",\"x\":\"JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs\",\"use\":\"sig\",\"nbf\":1712793600,\"exp\":1715385600,\"x5c\":[\"MIIBYTCCAQagAwIBAgIUFDXRG3pgZ6txehQO2LT4aCqI3f0wCgYIKoZIzj0EAwIwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wHhcNMjUwNjEzMTA0MjQxWhcNMzUwNjExMTA0MjQxWjAaMRgwFgYDVQQDDA9zdWIuZXhhbXBsZS5jb20wKjAFBgMrZXADIQAmtAuPk//z2JcRL368WCsjLb1yUX0IL+g8+zDdzkPRu6NdMFswCQYDVR0TBAIwADAOBgNVHQ8BAf8EBAMCB4AwHQYDVR0OBBYEFKV3qaYNFbzQB1QmN4sa13+t4RmoMB8GA1UdIwQYMBaAFFtwp5gX95/2N9L349xEbCEJ17vUMAoGCCqGSM49BAMCA0kAMEYCIQC8r+GvvNnjI+zzOEDMOM/g9e8QLm00IZXP+tjDqah1UQIhAJHffLke9iEP1pUdm+oRLrq6bUqyLELi5TH2t+BaagKv\",\"MIIBcDCCARagAwIBAgIUS502rlCXxG2vviltGdfe3fmX4pIwCgYIKoZIzj0EAwIwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wHhcNMjUwNjEzMTA0MTQzWhcNMzUwNjExMTA0MTQzWjAWMRQwEgYDVQQDDAtleGFtcGxlLmNvbTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABEIlSPiPt4L/teyjdERSxyoeVY+9b3O+XkjpMjLMRcWxbEzRDEy41bihcTnpSILImSVymTQl9BQZq36QpCpJQnKjQjBAMA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgIEMB0GA1UdDgQWBBRbcKeYF/ef9jfS9+PcRGwhCde71DAKBggqhkjOPQQDAgNIADBFAiEAwTOqm1zNAvZuQ8Zb5AftQIZotq4Xe6GHz3+nJ04ybgoCIEEZtn1Pa+GCbmbWh12piHJBKh09TCA0feTedisbwzPV\"]}]}"
{"hello": "world"}

HTTP/1.1 200 OK
{"message": "good dog"}
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

# Changelog
{:numbered="false"}

v03

- Remove "purpose" field from web-bot-auth

v02

- Correct typos

v01

- Update content-type from `application/http-message-signatures-directory` to `application/http-message-signatures-directory+json`
- Add delegation and chaining examples: full x5c chain, AIA extension, and x5u
- Add inline directory example with data URI
- Fix well-known path in examples

v00

- Initial draft
- Definition of Signature-Agent and its three supported URI https, http, and data.
- Leverages JWKS as a directory fo HTTP Message Signatures
- Well-known and content-type
