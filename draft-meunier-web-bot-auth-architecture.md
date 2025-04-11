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
 - not-yet
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
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  DIRECTORY:
    anchor: I-D.draft-meunier-httpbis-http-message-signatures-directory
    title: HTTP Message Signatures Directory
    target: https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-httpbis-http-message-signatures-directory.html

informative:


--- abstract

This document describes an architecture for identifying automated
traffic using {{HTTP-MESSAGE-SIGNATURES}}. The goal
is to allow automated HTTP clients to cryptographically sign outbound
requests, allowing HTTP servers to verify their identity with confidence.


--- middle

# Introduction

Automated agents are increasingly used for legitimate purposes on the web, including AI assistants,
search indexing, content aggregation, and automated testing. These agents need to reliably identify
themselves to origins for several reasons:

1. Regulatory compliance requiring transparency of automated systems
2. Origin resource management and access control
3. Protection against impersonation and reputation management
4. Service level differentiation between human and automated traffic

Current identification methods such as IP allowlisting, User-Agent strings, or shared API keys have
significant limitations in security, scalability, and manageability. This document defines an
architecture enabling automated agents to cryptographically identify themselves using {{HTTP-MESSAGE-SIGNATURES}}.
It proposes that every request from bots to be signed by a private key owned by its provider.
This way, every origin can validate the service identity.

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

## HTTP layer choice

This architectures operates solely at the HTTP layer.
It allows signatures to be generated and
verified without modifying the transport layer or TLS stack. It enables
flexible deployment across proxies, gateways, and origin servers, and aligns
with existing tooling and infrastructure that already inspect and manipulate
HTTP headers.

Because the signature is embedded in the request itself, it travels with the
message through intermediaries, preserving end-to-end verifiability even when
requests are forwarded or transformed within the HTTP layer.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

**User**
: An entity initiating requests through an automated agent. May be a human operator or another system.

**Automated Agent**
: A software system making HTTP requests on behalf of users. It implements the HTTP protocol and constructs valid HTTP requests with {{HTTP-MESSAGE-SIGNATURES}} signatures.

**Origin**
: An HTTP server receiving signed requests that implements the HTTP protocol and verifies {{HTTP-MESSAGE-SIGNATURES}} signatures. It acts as a verifier of the signature as defined by {{HTTP-MESSAGE-SIGNATURES}}.

# Architecture

> TODO: System diagram + one more sequence diagram with key retrieval from the Automated Agent provider

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

A User initiates an action requiring the Autoimated Agent to perform an HTTP request.
The Automated Agent constructs the request, generates a signature using its signing key,
and includes it in the request as defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}
along with `Signature-Agent` header for discovery for its
verification key.
Upon receiving the request, the Origin ensures it has the verification key for the Agent,
validates the signature, and processes the request if the signature is valid.

## Deployment Models

Signature verification can be performed either directly by origins or delegated to a fronting proxy. Direct verification by origins provides simplicity and control. Proxy verification offloads processing and enables shared caching across multiple origins. The choice depends on traffic volume and operational requirements.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURES}} defines components to be signed.

Automated agents SHOULD include the following component:

`@authority`
: as defined in {{Section 2.2.1 of HTTP-MESSAGE-SIGNATURES}}

Automated agents SHOULD include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`keyid`
: MUST be `SHA256(key_bytes)`

`tag`
: MUST be `web-bot-auth`

The private key is available to the automated agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.

We RECOMMEND the expiry to be no more than 24 hours.

### Anti-replay

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.

`@signature-parameters` are extended as follow

`nonce`
: Base10 encoded random uint32

This `nonce` MUST be unique for the validity window of the signature, as defined by created and expires attributes.
Because the `nonce` is controlled by the client, the origin needs to maintain a list of all nonces that it has seen that are still in the validity window of the signature.

## Requesting a Message signature

{{Section 5 of HTTP-MESSAGE-SIGNATURES}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin. Origin MAY choose to request signatures from clients that did not initially provide them. If requesting, origins MUST to request the same parameters as those defined by the {{generating-http-message-signature}}.

## Validating Message signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURES}}.
Similar to regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirement are placed on this validation

During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
During step 5, the Origin MAY discard signatures for which they do not know the `keyid`.
During step 5, if the keyid is unknown to the origin, they MAY fetch the provider directory as indicated by `Signature-Agent` header defined in Section 4 of {{DIRECTORY}}.


## Discovery

This section describes the discovery mechanism for the automated agent directory.

The reference for discovery is a FQDN. It SHOULD provide a directory hosted on the well known registered in Section 4 of {{DIRECTORY}}.

We add one field to the directory defined in the other draft:
"purpose": Ideally matches some IANA registry for preferences

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

## Performance Impact

Origins must account for the overhead of signature verification in their operations. A local cache of public keys reduces network requests and verification latency. The choice of signing algorithm impacts CPU requirements. Origins should monitor verification latency and set appropriate timeouts to maintain service levels under load.

## Key Compromise Response

An automated agent signing key might get compromised.

If that happens, the agent SHOULD cease using the compromised key as soon as possible, notify affected origins if possible, and generate a new key pair.

To minimise the impact of a key compromise, the origin should support rapid key rotation and monitor for suspicious signature patterns.

## Batching Signatures

To reduce signature frequency and improve efficiency, clients MAY batch
multiple operations into a single signed request, where semantically
appropriate. This technique can amortize the signing cost over multiple
actions, but it MUST NOT obscure the intent or structure of the request in a
way that undermines verifiability or introduces ambiguity.

## Shared Secrets Considered Harmful

Implementations MUST NOT use shared HMAC defined in {{Section 3.3.3 of HTTP-MESSAGE-SIGNATURES}}.
Shared secrets break non-repudiation and make auditing
difficult. Each automated client SHOULD use a unique asymmetric keypair to
ensure attribution, support key rotation, and enable effective revocation if
needed.

# Privacy Considerations

## Public Identity

This architecture assumes that automated clients identify themselves
explicitly using digital signatures. The identity associated with a signing
key is expected to be publicly discoverable for verification purposes. This
reduces anonymity and allows receivers to associate requests with specific
automated agents.

## No Human Correlation

The key used for signing MUST NOT be tied to a specific human individual.
Keys SHOULD represent a role or automation identity (e.g., "news-aggregator-
bot", "example-crawler-v1"). This avoids accidental exposure of personally
identifiable information and prevents the misuse of keys for user tracking or
profiling.

## Minimizing Tracking Risks

To limit tracking risks, implementations SHOULD avoid long-lived, globally
unique key identifiers unless strictly necessary. Key rotation SHOULD be
supported, and clients SHOULD take care to avoid signing information that
could be used to correlate activity across contexts, especially where
sensitive user data is involved.


# IANA Considerations

This document has no IANA actions.


--- back

# Implementations

> Thibault: add github link once open

This draft has a couple of implementations

Clients:

* Chrome MV3

* Cloudflare Workers

Servers:

* Caddy plugin

* Cloudflare Workers

A demontstration server has been deployed to https://http-message-signatures-example.research.cloudflare.com/
It uses RFC9421 ed25519 test signing and verifying keys.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
