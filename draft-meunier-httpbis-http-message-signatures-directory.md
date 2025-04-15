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
 - not-yet
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
  HTTP: RFC9110
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP-MESSAGE-SIGNATURES-IANA:
    title: HTTP Message Signatures
    target: https://www.iana.org/assignments/http-message-signature/http-message-signature.xhtml
  JWK: RFC7517
  STRUCTURED-HEADERS: RFC8941
  URI: RFC8820
  WellKnownURIs:
    title: Well-Known URIs
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml

informative:
  BASE64: RFC2397


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
1. A standardized key directory format based on JWKS for publishing HTTP Message Signatures keys
2. A well-known URI location for discovering these key directories
3. A new HTTP header field enabling in-band key directory location discovery

Together, these mechanisms enable key distribution and discovery for HTTP Message Signatures cryptographic material.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Configuration {#configuration}

The key directory is served as a JSON Web Key Set (JWKS) as defined in {{Section 5 of JWK}}.
The "alg" parameter are restricted to algorithm registered against HTTP Signature Algorithms Section of {{HTTP-MESSAGE-SIGNATURES-IANA}}

The directory SHOULD be server over HTTPS.
The directory MUST be served with media type `application/http-message-signatures-directory`.

Client application SHOULD validate the directory format and reject malformed entries.

# HTTP Method Context `Signature-Agent`

A service sending signed requests as defined in {{HTTP-MESSAGE-SIGNATURES}} MAY include a
`Signature-Agent` header field to communicate its signing key directory. This header
field contains a URI allowing retrieval of an HTTP Message Signatures Directory as
defined in {{configuration}}.

## Header Field Definition

The `Signature-Agent` header field is an Item Structured Header {{STRUCTURED-HEADERS}}. Its value MUST be a
String containing a {{URI}}. The ABNF is:

~~~
Signature-Agent = sf-string   ; Section 3.3.3 of {{STRUCTURED-HEADERS}}
~~~

The URI scheme MUST be one of:
- **https (RECOMMENDED)**: Points to an HTTPS resource serving the key directory
- **http**: Points to an HTTP resource serving the key directory
- **data**: Contains an inline key directory

When using the "data" URI scheme, the media type MUST be
`application/http-message-signatures-directory`. The content MAY be base64 encoded
as per {{BASE64}}.

Multiple `Signature-Agent` header fields MAY be present in a request. Processors SHOULD
use the first valid URI that provides a valid key directory.

TODO: for inlining, CBOR Keys could be useful

# Security Considerations {#security}

## Key rotation

Clients SHOULD implement key rotation by including multiple keys in the directory
with different validity period. When rotating keys, clients SHOULD:

1. Add the new key to the directory before its intended use date
2. Continue to include the old key until its expiration date
3. Remove expired keys from the directory

Servers SHOULD cache the directory contents and refresh upon expiration.

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

This extend the examples from {{Appendix B of HTTP-MESSAGE-SIGNATURES}}.

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
