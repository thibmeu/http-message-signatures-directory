---
title: "HTTP Message Signatures for automated traffic Architecture"
abbrev: "HTTP Message Signatures for Bots"
category: info

docname: draft-meunier-web-bot-auth-architecture-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Web and Internet Transport
workgroup: Web Bot Auth
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Web Bot Auth
  mail: "web-bot-auth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-web-bot-auth-architecture.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  HTTP-MESSAGE-SIGNATURE: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  DIRECTORY: I-D.draft-meunier-httpbis-http-message-signatures-directory

informative:


--- abstract

This document describes an architecture for identifying automated
traffic using {{HTTP-MESSAGE-SIGNATURE}}. The goal
is to allow automated HTTP clients to cryptographically sign outbound
requests, allowing HTTP servers to verify their identity with confidence.


--- middle

# Introduction

Automated agents are increasingly used for legitimate
purposes, including AI assistants, indexing, testing, and automated
support tools. These agents often use general-purpose browsers,
making it difficult for websites to authenticate or identify them.
Existing methods, such as IP range allowlisting or User-Agent
strings, offer no integrity guarantees and are hard to maintain.

This document proposes a mechanism in which outbound HTTP requests
are signed using {{HTTP-MESSAGE-SIGNATURE}}. These signatures
can be verified by receiving servers using a public key associated
with the platform provider. This enables trusted interactions between
automated agents and HTTP servers, with improved security and
manageability.

# Motivation

There is an increase in automated agents on the Internet. Some are honest, and want to identify themselves.
Either because origins pressure them to do so, because regulations mandate transparency, or because they are
facing an increased phishing threat. As of today these automated agents are left without any good solution to do so:

1. They share their IP range. However that is not sustainable because the same IP might be used by different
   services, IP ranges may change, geolocation imposes to register IPs in multiple countries, and when they start
   allowing other companies to use their platform they loose control of their public facing reputation.
2. They define a User Agent per {{Section 10.1.5 of HTTP}}. Like curl uses `curl/version`, or Chrome uses
   `Mozilla/5.0 ... Chrome/113.0.0.0`. An issue is this header is spoofable, and realistically automated agents are
   likely to use Chrome's user agent because otherwise they are challenged
3. They go to every website on the Internet and share a secret with them like a Bearer from  {{!RFC6750}}. This is
   impractical at scale.

All this provides strong motivation to define a mechanism that empowers honest automated agents to share their identity.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

**User**
: wants to perform some actions on the web.

**Automated Agent**
: orchestrated user agent, such as Chromium. Can interact with the web, implements web standards. For every request, it constructs a valid HTTP request with {{HTTP-MESSAGE-SIGNATURE}} signature.

**Origin**
: server hosting a resource. The user wants to access it through the browser.

# Architecture

> TODO: Add directory

~~~aasvg
  +---------+             +-----------------+             +--------+
  |  User   |             | Automated Agent |             | Origin |
  +----+----+             +--------+--------+             +---+----+
       |                          |                           |
       |  Help me do this!        |                           |
       +------------------------->|                           |
       |                          |  GET /path/to/resource    |
       |                          |  Signature: abc===        |
       |                          +-------------------------->|
       |                          |                           |
       |                          |     <h1>Response</h1>     |
       |                          |<--------------------------+
       |  Here are proposals      |                           |
       +<-------------------------+                           |
       |                          |                           |

~~~

A user is asking the automated agent to do something. The agent decides they
need to retrieve content from an origin. They send an HTTP request with a Signature
header as defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURE}}.
The origin validates the signature. If it validates, it sends the requested content.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURE}} defines components to be signed.

Automated agents SHOULD include the following component:

`@authority`
: as defined in {{Section 2.2.1 of HTTP-MESSAGE-SIGNATURE}}

Automated agents SHOULD include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURE}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURE}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURE}}

`keyid`
: MUST be `SHA256(key_bytes)`

`tag`
: MUST be `web-bot-auth`

The private key is available to the automated agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURE}}.

### Anti-replay

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.

`@signature-parameters` are extended as follow

`nonce`
: Base10 encoded random uint32

This `nonce` MUST be unique for the validity window of the signature, as defined by created and expires attributes.
Because the `nonce` is controlled by the client, the origin needs to maintain a list of all nonces that it has seen that are still in the validity window of the signature.

## Requesting a Message signature

{{Section 5 of HTTP-MESSAGE-SIGNATURE}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin. Origin MAY choose to request signatures from clients that did not initially provide them. If requesting, origins MUST to request the same parameters as those defined by the {{generating-http-message-signature}}.

## Validating Message signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURE}}.
Similar to regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirement are placed on this validation

During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
During step 5, the Origin MAY discard signatures for which they do not know the `keyid`.
During step 5, if the keyid is unknown to the origin, they MAY fetch the provider directory as indicated by `Signature-Agent` header defined in {{Section 4 of DIRECTORY}}.


## Discovery

This section describes the discovery mechanism for the automated agent directory.

The reference for discovery is a FQDN. It SHOULD provide a directory hosted on the well known registered in {{Section 4 of DIRECTORY}}.

We add one field to the directory defined in the other draft:
"purpose": Ideally matches some IANA registry for preferences

TODO: replace the key with a JWK

Example

~~~json
{
  "keys": {
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  },
  "signature_agent": "https://directory.test",
  "purpose": "rag"
}
~~~


### Out-of-band communication between client and origin
A service submitting their key to an origin, or the origin manually adding a service to their trusted list

### Public list

Could be a GitHub repository like the public suffix list. The issue is the gating of such repositories, and therefore its governance.

### Signature-Agent header

This is defined in the sibling draft.
This allows for backward compatibility with existing header agent filtering, and an upgrade to cryptographically secured protocol.


# Security Considerations

Verification load
Bigger request

> Hint at req mTLS

# Privacy Considerations

Identity is public


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

# Implementations
{:numbered="false"}

This draft has a couple of implementations

Clients:

* Chrome MV3

* Cloudflare Workers

Servers:

* Caddy plugin

* Cloudflare Workers

A demontstration server has been deployed to https://http-message-signatures-example.research.cloudflare.com/
It uses RFC9421 ed25519 test signing and verifying keys.
