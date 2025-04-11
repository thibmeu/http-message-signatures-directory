---
title: "HTTP Message Signatures Directory"
abbrev: "HTTP Message Signatures Directory"
category: std

docname: draft-meunier-httpbis-http-message-signatures-directory-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "HTTP"
  type: ""
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-httpbis-http-message-signatures-directory.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  HTTP-MESSAGE-SIGNATURE: RFC9421
  HTTP: RFC9110
  JWK: RFC7517
  WellKnownURIs:
    title: Well-Known URIs
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml
  HTTP-MESSAGE-SIGNATURES-IANA:
    title: HTTP Message Signatures
    target: https://www.iana.org/assignments/http-message-signature/http-message-signature.xhtml

informative:


--- abstract

This document describe a method for clients using using {{HTTP-MESSAGE-SIGNATURE}}
to advertise their signing keys.

It defines a key directory format based on JWKS as defined in {{Section 5 of JWK}},
as well as new HTTP Method Context to allow for in-band key discovery.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Configuration {#configuration}

The key directory is served as a JSON Web Key Set (JWKS) as defined in {{Section 5 of JWK}}.
The "alg" parameter are restricted to algorithm registered against {{HTTP Signature Algorithms Section of HTTP-MESSAGE-SIGNATURES-IANA}}

The directory SHOULD be server over HTTPS.
The directory MUST be served with media type `application/http-message-signatures-directory`.

Client application SHOULD validate the directory format and reject malformed entries.

# HTTP Method Context `Signature-Agent`

A service sending signed request as defined in {{HTTP-MESSAGE-SIGNATURE}} MAY
provide HTTP Method Context `Signature-Agent`.
This header is defined as a URI to a directory.

The URI scheme SHOULD be one of:

1. http
2. https
3. data

http/https point to the origin component serving an HTTP Message Signatures as defined in {{configuration}}.
data MUST have media type `application/http-message-signatures-directory`, and MAY be base64 encoded. (
CBOR would be very useful here)

This extends the set of headers defined in {{!RFC9110}}.

Signature-Agent

```
Signature-Agent    = [[ URI ]]
```

# Security Considerations {#security}

## Key rotation

> TODO: reference key directory over http draft

Clients SHOULD implement key rotation by including multiple keys in the directory
with different validity period. When rotating keys, clients SHOULD:

1. Add the new key to the directory before its intended use date
2. Continue to include the old key until its expiration date
3. Remove expired keys from the directory

Servers SHOULD cache the directory contents and refresh upon expiration.


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

- "application/http-message-signatures-directory"

The templates for these entries are listed below and the
reference should be this RFC.

### "application/http-message-signatures-directory" media type

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
GET /.well-known/bar HTTP/1.1
Host: example.com
Accept: application/http-message-signatures-directory

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory
Cache-Control: max-age=86400
{
  "keys": {
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  }
}
~~~

## Request with HTTP Signature-Agent

This extend the examples from {{Appendix B of HTTP-MESSAGE-SIGNATURE}}.

~~~
POST /foo?param=Value&Pet=dog HTTP/1.1
Host: example.com
Signature-Agent: https://directory.test
{"hello": "world"}

HTTP/1.1 200 OK
{"message": "good dog"}
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
