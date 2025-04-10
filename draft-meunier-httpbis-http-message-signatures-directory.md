---
title: "HTTP Message Signatures Directory"
abbrev: "HTTP Message Signatures Directory"
category: info

docname: draft-meunier-httpbis-http-message-signatures-directory-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
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

# Configuration

> TODO: should there be a header parameter to accept JSON or CBOR?

Service that aim to sign their request using {{HTTP-MESSAGE-SIGNATURE}}  can share their public
key as JWKS

> TODO: replace with actual JWKS

{
  keys: [{
    alg:"ed25519",
    key: "-----BEGIN PUBLIC KEY-----...",
    not-before: 1743578485, // optional
    not-after: 1745000000, // optional
  }]
}

# Discoverability

A service sending signed request as defined in {{HTTP-MESSAGE-SIGNATURE}} MAY
send the FQDN hosting its directory via a new HTTP Method Context `Signature-Agent`.

This extends the set of headers defined in {{!RFC9110}}.

Signature-Agent

```
Signature-Agent    = fqdn

fqdn = <fqdn, see [RFC9110], Section XX>
```

# Security Considerations {#security}

TODO Security


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

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
