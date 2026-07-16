---
title: "HTTP Message Signatures Directory"
abbrev: "HTTP Message Signatures Directory"
category: std

docname: draft-meunier-webbotauth-httpsig-directory-latest
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
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-webbotauth-httpsig-directory.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk
 -
    fullname: Sandor Major
    organization: Google
    email: ietf@sandormajor.com

normative:
  CIMD: I-D.draft-ietf-oauth-client-id-metadata-document
  DIGEST-FIELDS: RFC9530
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


--- abstract

This document describes a method for clients using {{HTTP-MESSAGE-SIGNATURES}}
to advertise their signing keys.

It defines a key directory format based on JWKS as defined in {{Section 5 of JWK}},
as well as a new HTTP Method Context for in-band key discovery.


--- middle

# Introduction

{{HTTP-MESSAGE-SIGNATURES}} allow a signer to generate a signature over an HTTP message, and a verifier to validate it.
The specification assumes verifiers have prior knowledge of
signers' key material, requiring out-of-band key distribution mechanisms. This creates deployment
friction and limits the ability to dynamically verify signatures from previously unknown signers.

This document defines:

1. A standardized key directory format based on JWKS for publishing HTTP Message Signatures keys,
2. A well-known URI location for discovering these key directories,
3. A new HTTP header field enabling in-band key material discovery.

Together, these mechanisms enable key distribution and discovery for HTTP Message Signatures cryptographic material.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Configuration {#configuration}

The key directory is served as a JSON Web Key Set (JWKS) as defined in {{Section 5 of JWK}}.
The "alg" parameter are restricted to algorithm registered against HTTP Signature Algorithms Section of {{HTTP-MESSAGE-SIGNATURES-IANA}}

The directory MUST be served over HTTPS.
The directory MUST be served with media type `application/http-message-signatures-directory+json`.

A verifier SHOULD validate the directory format and reject malformed entries.

# HTTP Method Context `Signature-Agent`

A service sending signed requests as defined in {{HTTP-MESSAGE-SIGNATURES}} MAY include a
`Signature-Agent` header field to communicate where its verification key material can be found.
This header field contains a URI and a `type` parameter that identifies the discovery mechanism.

## Header Field Definition

The `Signature-Agent` header field is a Dictionary Structured Header as defined
in {{Section 3.2 of STRUCTURED-HEADERS}}.
Its member values MUST be String Items that contain a {{URI}}.

The `type` parameter is a Token Item as defined in {{Section 3.3.4 of
STRUCTURED-HEADERS}}. If the `type` parameter is absent, its value is `directory`.

The following `type` values are defined:

`directory`
: The member value identifies an origin. A verifier resolves the HTTP Message
Signatures Directory using the well-known URI registered in {{wkuri-reg}} at
that origin.

`jwks_uri`
: The member value identifies a JWK Set URI.

`cimd`
: The member value identifies a Client ID Metadata Document {{CIMD}} URI.

A verifier that does not support a `type` value MUST ignore that member.
A verifier MUST NOT infer the discovery mechanism from the URI path, media type,
or response body.

[[ Editor's note: strengthen requirements around jwks_uri and cimd. directory
uses a signed well-known to guarantee {{security}} ]]

The URI scheme MUST be `https`.

[[ Editor's note: earlier versions also allowed `http` and `data` URI
schemes. A `data` URI carries inline key material with no authority behind
it, which muddied what the directory binding proves. Both are removed for now. ]]

If dictionary values are not valid URI-references, the entire header field MAY be
ignored.

# Security Considerations {#security}

## Key rotation

Directory operators SHOULD implement key rotation by including multiple keys
in the directory with a different validity period. When rotating keys,
operators SHOULD:

1. Add the new key to the directory before its intended use date
2. Continue to include the old key until its expiration date
3. Remove expired keys from the directory

Verifiers SHOULD cache the directory contents and refresh upon expiration.

Removing a key from the directory deactivates it. Verifiers stop accepting keys
once their cached copy expires. The directory's cache lifetime therefore bounds
how long a removed key keeps verifying.

[[ Editor's note: Recommendation around lifetime? ]]

## Binding keys to the directory authority

To ensure the authenticity and integrity of the key material provided by the
directory, verifiers SHOULD validate the directory's response.

It is RECOMMENDED that a directory server construct and include one HTTP
Message Signature per key with the response, as defined in
{{HTTP-MESSAGE-SIGNATURES}}.
Each key SHOULD be used to provide one signature. These signatures prove
possession of the advertised keys and, by covering `@authority`, prevent the
key set from being re-served under a different authority. They do not confer
authority over the serving domain: that comes from the TLS connection alone.

Directory server SHOULD include the following covered components:

`@authority`
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}. `req` flag defined in {{Section 2.4 of HTTP-MESSAGE-SIGNATURES}} MUST be set.

`content-digest`
: as defined in {{DIGEST-FIELDS}}.

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

Verifiers relying on the domain binding MUST validate these signatures
using the keys provided by the directory, validate the `Content-Digest`
field against the response body, and ignore keys that do not have a
corresponding valid signature. Verifiers using keys opaquely MAY skip this
validation, in which case they obtain no binding. This validation checks
the integrity of the key set and binds it to the intended authority.

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
| http-message-signatures-directory | IETF | this document | permanent | None |
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

Delegation and chaining are out of scope for this document and are expected
to be specified separately. Input is welcome on the associated
[GitHub issue](https://github.com/thibmeu/http-message-signatures-directory/issues/27).

[[ Editor's note: earlier versions carried informative delegation examples
using `x5c`, AIA, and `x5u`. They suggested verifier behaviour this document
does not define, so they are removed for clarity purposes. Issue 27 tracks
a possible separate document. ]]

## Request with HTTP Signature-Agent

This extend the examples from {{Appendix B of HTTP-MESSAGE-SIGNATURES}}.

~~~
POST /foo?param=Value&Pet=dog HTTP/1.1
Host: example.com
Signature-Agent: my_test="https://directory.test";type=directory
{"hello": "world"}

HTTP/1.1 200 OK
{"message": "good dog"}
~~~

# Acknowledgments
{:numbered="false"}

Marwan Fayed,
Maxime Guerreiro,
Jonathan Hoyland,
Nikhil Kandoi,
Akshat Mahajan,
Eugenio Panero,
Lucas Pardue.

# Changelog
{:numbered="false"}

draft-meunier-webbotauth-httpsig-directory-01

- Tighten the domain binding: `https` only, MUST validate when relying on it.
- Remove delegation examples; use verifier and directory operator role terms.

draft-meunier-webbotauth-httpsig-directory-00

- Rename draft from `draft-meunier-http-message-signatures-directory`.
- Add typed `Signature-Agent` discovery with `directory`, `jwks_uri`, and
  `cimd`; default to `directory` when no type is present.
- Add `Content-Digest` to directory response validation.
- Update `Signature-Agent` examples to use `type=directory`.
- Fix wording around URI values and directory response signatures.

draft-meunier-http-message-signatures-directory-05

- Add Sandor Major as an author.
- Remove a stale author TODO from the examples.

draft-meunier-http-message-signatures-directory-04

- Change `Signature-Agent` to a Structured Fields dictionary.
- Add the `req` flag on directory response signatures.
- Add contributors.

draft-meunier-http-message-signatures-directory-03

- Remove the `purpose` field from the Web Bot Auth example.

draft-meunier-http-message-signatures-directory-02

- Fix typos.

draft-meunier-http-message-signatures-directory-01

- Change the media type from `application/http-message-signatures-directory` to
  `application/http-message-signatures-directory+json`.
- Add delegation and chaining examples using a full `x5c` chain, AIA, and
  `x5u`.
- Add an inline directory example using a `data` URI.
- Fix the well-known path in examples.

draft-meunier-http-message-signatures-directory-00

- Initial draft.
- Define `Signature-Agent` for `https`, `http`, and `data` URI values.
- Use JWKS as a directory for HTTP Message Signatures keys.
- Define the well-known URI and media type.
