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
  DIRECTORY: I-D.draft-meunier-http-message-signatures-directory
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638


informative:
  OAUTH-BEARER: RFC6750

--- abstract

This document describes an architecture for identifying automated
traffic using {{HTTP-MESSAGE-SIGNATURES}}. The goal
is to allow automated HTTP clients to cryptographically sign outbound
requests, allowing HTTP servers to verify their identity with confidence.


--- middle

# Introduction

Agents are increasingly used in business and user workflows, including AI assistants,
search indexing, content aggregation, and automated testing. These agents need to reliably identify
themselves to origins for several reasons:

1. Regulatory compliance requiring transparency of automated systems
2. Origin resource management and access control
3. Protection against impersonation and reputation management
4. Service level differentiation between human and automated traffic

Current identification methods such as IP allowlisting, User-Agent strings, or shared API keys have
significant limitations in security, scalability, and manageability. This document defines an
architecture enabling agents to cryptographically identify themselves using {{HTTP-MESSAGE-SIGNATURES}}.
It proposes that every request from bots to be signed by a private key owned by its provider.
This way, every origin can validate the service identity.

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
    the publication to the actual service provider that may be renting infra. Purchasing
    dedicated IP blocks is expensive, time consuming, and requires significant
    specialist knowledge to set up. These IP blocks may have prior reputation
    history that needs to be carefully inspected and managed before purchase and
    use.
 3. An agent may go to every website on the Internet and share a secret with
    them like a Bearer from  {{OAUTH-BEARER}}. This is impractical to scale for any
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
: An entity initiating requests through an agent. May be a human operator or another system.

**Agent**
: An orchestrated user agent (e.g. Chromium, CURL). It implements the HTTP protocol and constructs valid HTTP requests with {{HTTP-MESSAGE-SIGNATURES}} signatures.

**Origin**
: An HTTP server receiving signed requests that implements the HTTP protocol and verifies {{HTTP-MESSAGE-SIGNATURES}} signatures. It acts as a verifier of the signature as defined by {{HTTP-MESSAGE-SIGNATURES}}.

# Architecture

~~~aasvg
+--------+               +---------+                           +----------+
|        |               |         |         Exchange          |          |
|        |               |         |<===== Cryptographic =====>|          |
|        |               |         |         material          |          |
|  User  +--- Request -->|  Agent  |                           |  Origin  |
|        |               |         +--- Request + Signature -->|          |
|        |               |         |<-------- Response --------+          |
|        |<-- Response --+         |                           |          |
|        |               |         |                           |          |
+--------+               +---------+                           +----------+
~~~

A User initiates an action requiring the Agent to perform an HTTP request.
The Agent constructs the request, generates a signature using its signing key,
and includes it in the request as defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}
along with `Signature-Agent` header for discovery for its
verification key.
Upon receiving the request, the Origin ensures it has the verification key for the Agent,
validates the signature, and processes the request if the signature is valid.

## Deployment Models

Signature verification can be performed either directly by origins or delegated to a fronting proxy. Direct verification by origins provides simplicity and control. Proxy verification offloads processing and enables shared caching across multiple origins. The choice depends on traffic volume and operational requirements.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURES}} defines components to be signed.

Agents MUST include the following component:

`@authority`
: as defined in {{Section 2.2.1 of HTTP-MESSAGE-SIGNATURES}}

Agents MUST include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`keyid`
: MUST be a base64url JWK SHA-256 Thumbprint as defined in {{Section 3.2 of JWK-THUMBPRINT}} for RSA and EC, and in {{Appendix A.3 of JWK-OKP}} foe ed25519.

`tag`
: MUST be `web-bot-auth`

The signing key is available to the agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.

It is RECOMMEND the expiry to be no more than 24 hours.

### Anti-replay

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.

Agents SHOULD extend `@signature-parameters` defined in {{generating-http-message-signature}} as follow

`nonce`
: base64url encoded random byte array. It is RECOMMENDED to use a 64-byte array.

This `nonce` MUST be unique for the validity window of the signature, as defined by created and expires attributes.
Because the `nonce` is controlled by the client, the origin needs to maintain a list of all nonces that it has seen that are still in the validity window of the signature.

### Sending a request

An Agent SHOULD send a request with the signature generated above. Updating the architecture diagram, the flow look as follow.

~~~aasvg
+---------+                                                                     +----------+
|         |                              Exchange                               |          |
|         |<=========================  Cryptographic  =========================>|          |
|         |                              material                               |          |
|  Agent  |                                                                     |  Origin  |
|         |     .------------------------------------------------------.        |          |
|         +-----| GET /path/to/resource                                |------->|          |
|         |     | Signature: abc==                                     |        |          |
+---------+     | Signature-Input: sig=(@authority);tag="web-bot-auth" |        +----------+
                | Signature-Agent: signer.example.com                  |
                '------------------------------------------------------'
~~~

The Agent SHOULD send requests with two headers

1. `Signature` defined in {{generating-http-message-signature}}
2. `Signature-Input` defined in {{generating-http-message-signature}}

Mentioned in a section below, the Agent MAY send request with an additional header
3. `Signature-Agent` defined in {{signature-agent}}

## Requesting a Message signature {#requesting-message-signature}

{{Section 5 of HTTP-MESSAGE-SIGNATURES}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin. Origin MAY choose to request signatures from clients that did not initially provide them. If requesting, Origins MUST use the same parameters as those defined by the {{generating-http-message-signature}}.

Origin MAY request a new signature with tag "web-bot-auth" even if a nonce is provided, for example if it believes the nonce is a replay, or if it doesn't store nonces and thus requests new signatures every time.

## Validating Message signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURES}}.
Similar to regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirements are placed on this validation:

- During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
- During step 5, the Origin MAY discard signatures for which they do not know the `keyid`.
- During step 5, if the keyid is unknown to the origin, they MAY fetch the provider directory as indicated by `Signature-Agent` header defined in Section 4 of {{DIRECTORY}}.


## Key Distribution and Discovery

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

### Signature-Agent header {#signature-agent}

This is defined in the sibling draft.
This allows for backward compatibility with existing header agent filtering, and an upgrade to cryptographically secured protocol.

# Security Considerations

## Performance Impact

Origins must account for the overhead of signature verification in their operations. A local cache of public keys reduces network requests and verification latency. The choice of signing algorithm impacts CPU requirements. Origins should monitor verification latency and set appropriate timeouts to maintain service levels under load.

## Key Compromise Response

An agent signing key might get compromised.

If that happens, the agent SHOULD cease using the compromised key as soon as possible, notify affected origins if possible, and generate a new key pair.

To minimise the impact of a key compromise, the origin should support rapid key rotation and monitor for suspicious signature patterns.

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
Keys SHOULD represent a role, company, or automation identity (e.g., "news-aggregator-
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

This draft has a couple of [implementations](https://github.com/cloudflareresearch/web-bot-auth)

Clients:

* Chrome MV3

* Cloudflare Workers

Servers:

* Caddy plugin

* Cloudflare Workers

A demontstration server has been deployed to [https://http-message-signatures-example.research.cloudflare.com/](https://http-message-signatures-example.research.cloudflare.com/).

It uses ed25519 example signing and verifying keys defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
