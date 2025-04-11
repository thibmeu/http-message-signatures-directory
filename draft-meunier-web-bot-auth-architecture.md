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
 -
    fullname: Scott Hendrickson
    organization: Google
    email: scott@shendrickson.com

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

Agents are increasingly used in buisness and user workflows, including
AI assistants, indexing, testing, and automated support tools.  Existing methods
for sites to differentiate these workflows, such as IP range allowlisting or
User-Agent strings, offer no integrity guarantees and are hard to maintain.

This document proposes a mechanism in which outbound HTTP requests
are signed using {{HTTP-MESSAGE-SIGNATURES}}. These signatures
can be verified by receiving servers using a public key associated
with the platform provider. This enables trusted interactions between
agents and HTTP servers, with improved security and
manageability.

# Motivation

There is an increase in agent traffic on the Internet. Many agents
choose to identify their traffic today via IP Address lists and/or unique
User-Agents. This is often done to demonstrate trust and safety claims, support
allowlisting/denylisting the traffic in a granular manor, and enable sites to
monitor and rate limit per agent operator. However, these mechanisms have drawbacks:
 1. User-Agent, when used alone, can be spoofed meaning anyone may attempt to
    act as that agent. It is also overloaded - an agent may be using Chromium and
    wish to present itself as such to ensure rendering works, yet it still wants to
    differentiate its traffic to the site.
 2. IP blocks alone can present a confusing story. IPs on cloud plaforms have
    layers of ownership - the platform owns the IP and registers it in their
    published IP blocks, only to be re-published by the agent with little to bind
    the publication to (TODO: better example of PAAS layering?). Purchasing
    dedicated IP blocks is expensive, time consuming, and requires significant
    specialist knowledge to set up. These IP blocks may have prior reputation
    history that needs to be carefully inspected and managed before purchase and
    use.
 3. An agent may go to every website on the Internet and share a secret with
    them like a Bearer from  {{!RFC6750}}. This is impractical to scale for any
    agent beyond select partnerships, and insecure, as key rotation is challenging
    and becomes less secure as the consumers scale.

Using well-established cryptography, we can instead define a simple and secure
mechanism that empowers small and large agents to share their identity.

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
: wants to perform some actions on the web.

**Agent**
: orchestrated user agent (e.g. Chromium, CURL). Can interact with the web,
implements web standards. For every request, it constructs a valid HTTP request
with {{HTTP-MESSAGE-SIGNATURES}} signature.

**Origin**
: server hosting a resource. The user wants to access it through the browser.

# Architecture

> TODO: System diagram + one more sequence diagram with key retrieval from the Agent provider

~~~aasvg
  +---------+             +-----------------+             +--------+
  |  User   |             | Agent |             | Origin |
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

A user is asking the agent to do something. The agent decides they
need to retrieve content from an origin. They send an HTTP request with a Signature
header as defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.
The origin validates the signature. If it validates, it sends the requested content.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURES}} defines components to be signed.

Agents SHOULD include the following component:

`@authority`
: as defined in {{Section 2.2.1 of HTTP-MESSAGE-SIGNATURES}}

Agents SHOULD include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`keyid`
: MUST be `SHA256(key_bytes)`

`tag`
: MUST be `web-bot-auth`

The private key is available to the agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.

### Anti-replay

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.

`@signature-parameters` are extended as follow

`nonce`
: Base10 encoded random uint32

This `nonce` MUST be unique for the validity window of the signature, as defined by created and expires attributes.
Because the `nonce` is controlled by the client, the origin needs to maintain a list of all nonces that it has seen that are still in the validity window of the signature.

## Requesting a Message signature {#requesting-message-signature}

{{Section 5 of HTTP-MESSAGE-SIGNATURES}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin. Origin MAY choose to request signatures from clients that did not initially provide them. If requesting, origins MUST request the same parameters as those defined by the {{generating-http-message-signature}}.

## Validating Message signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURES}}.
Similar to regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirements are placed on this validation:

 - During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
 - During step 5, the Origin MAY discard signatures for which they do not know the `keyid`.
 - During step 5, if the keyid is unknown to the origin, they MAY fetch the provider directory as indicated by `Signature-Agent` header defined in Section 4 of {{DIRECTORY}}.


## Discovery

This section describes the discovery mechanism for the agent directory.

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

## Verification Load

Verifiers SHOULD be prepared to handle increased computational cost compared to
bearer verification or User-Agent parsing. Signature verification, particularly
with asymmetric keys, adds cryptographic overhead that may impact latency or
throughput. Implementers SHOULD provision resources accordingly and consider
other mechanisms to mitigate abuse. Implementers SHOULD only request message
signatures they intend on verifying as defined in {{#requesting-message-signature}}.

## Request Size Overhead

{{HTTP-MESSAGE-SIGNATURES}} include additional HTTP headers, such as the `Signature` and
`Signature-Input` headers, which increases overall request size on the wire. This may
affect intermediaries. HTTP clients and servers SHOULD monitor the impact
of larger request headers on routing and performance.

## Batching Signatures

TODO(scott): determine whether this makes sense to state. We don't talk a lot about the issuance in this draft.

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

## Key Reuse Considered Harmful

Implementations MUST NOT reuse a signing key for different purposes. For
example, if an agent implementor has two agents they want to differentiate,
these should use distinct signing keys and signing key directories.

# Privacy Considerations

## Public Identity

This architecture assumes that automated clients identify themselves
explicitly using digital signatures. The identity associated with a signing
key is expected to be publicly discoverable for verification purposes. This
reduces anonymity and allows receivers to associate requests with specific
agents. If an agent wishes not to identify itself, this is not the right
choice of protocol for it.

## No Human Correlation

The key used for signing MUST NOT be tied to a specific human individual.
Keys SHOULD represent a role, company or automation identity (e.g., "news-aggregator-
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
