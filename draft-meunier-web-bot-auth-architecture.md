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
  HTTP-MORE-STATUS-CODE: RFC6585
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638


informative:
  OAUTH-BEARER: RFC6750
  RFC8446:

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
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}

Agents MUST include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`keyid`
: MUST be a base64url JWK SHA-256 Thumbprint as defined in {{Section 3.2 of JWK-THUMBPRINT}} for RSA and EC, and in {{Appendix A.3 of JWK-OKP}} for ed25519.

`tag`
: MUST be `web-bot-auth`

The signing key is available to the agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.

It is RECOMMEND the expiry to be no more than 24 hours.

### Signature-Agent {#signature-agent}

`Signature-Agent` is an HTTP Method context header defined in Section 4.1 of {{DIRECTORY}}.
It is RECOMMENDED that the Agent sends requests with `Signature-Agent` header, as described in {{sending-request}}.
If the header is to be sent, it MUST be signed as a component as defined in {{Section 2.1 of HTTP-MESSAGE-SIGNATURES}}.

This results in the following components to be signed

~~~
("@authority" "signature-agent")
~~~

### Anti-replay {#anti-replay}

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.
This is described in {{Section 7.2.2 of HTTP-MESSAGE-SIGNATURES}}.

Agents SHOULD extend `@signature-parameters` defined in {{generating-http-message-signature}} as follow

`nonce`
: base64url encoded random byte array. It is RECOMMENDED to use a 64-byte array.

Client MUST ensure that this `nonce` is unique for the validity window of the signature, as defined by created and expires attributes.

### Additional headers

Agents MAY include additional components, such as specific HTTP headers, in the signature.
This can be prompted by the origin requesting additional headers, as described in {{requesting-message-signature}},
or initiated by the agent to provide more information within the signature scope.
For example, an agent might include an HTTP header expressing its intent and sign it.

Origins MAY ignore certain headers at their own discretion,
and request a new signature, as described in {{requesting-message-signature}}.

### Sending a request {#sending-request}

An Agent SHOULD send a request with the signature generated above. Updating the architecture diagram, the flow looks as follow.

~~~aasvg
+---------+                                                                    +----------+
|         |                              Exchange                              |          |
|         |<=========================  Cryptographic  ========================>|          |
|         |                              material                              |          |
|  Agent  |                                                                    |  Origin  |
|         |     .-----------------------------------------------------.        |          |
|         +-----| GET /path/to/resource                               |------->|          |
|         |     | Signature: abc==                                    |        |          |
+---------+     | Signature-Input: sig=(@authority signature-agent);\ |        +----------+
                |                  created=1700000000;\               |
                |                  expires=1700011111;\               |
                |                  keyid="ba3e64==";\                 |
                |                  tag="web-bot-auth"                 |
                | Signature-Agent: "https://signer.example.com"       |
                '-----------------------------------------------------'
~~~

The Agent SHOULD send requests with two headers

1. `Signature` defined in {{generating-http-message-signature}}
2. `Signature-Input` defined in {{generating-http-message-signature}}

Mentioned in {{signature-agent}}, the Agent MAY send requests with `Signature-Agent` header.

## Requesting a Message signature {#requesting-message-signature}

{{Section 5 of HTTP-MESSAGE-SIGNATURES}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin.
An Origin MAY choose to request signatures from clients that did not initially provide them. If requesting, Origins MUST use the same parameters as those defined by the {{generating-http-message-signature}}.
The status code SHOULD be 403 Forbidden as defined in {{Section 15.5.4 of HTTP}}.

Origin MAY request a new signature with tag "web-bot-auth" even if a nonce is provided, for example if it believes the nonce is a replay, or if it doesn't store nonces and thus requests new signatures every time.
The status code SHOULD be 429 Too Many Requests as defined in {{Section 4 of HTTP-MORE-STATUS-CODE}}.

## Validating Message signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURES}}.
Similar to a regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirements are placed on this validation:

- During step 1 to 3 included, if the Origin fails to parse the provided `Signature`, `Signature-Input`, or `Signature-Agent` headers, it MAY respond with status code 400 Bad Request as defined in {{Section 15.5.1 of HTTP}}.
- During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
- During step 5, the Origin MAY discard signatures for which they do not know the `keyid`.
- During step 5, if the keyid is unknown to the origin, they MAY fetch the provider directory as indicated by `Signature-Agent` header defined in Section 4 of {{DIRECTORY}}.

Origin MAY require the `nonce` to satisfy certain constraint: be globally unique using a global nonce store, be unique to a specific location or time window using a local cache, or not constraint at all.


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
A service submitting their key to an origin, or the origin manually adding a service to their trusted list.

### Public list

Could be a GitHub repository like the public suffix list. The issue is the gating of such repositories, and therefore its governance.

### Signature-Agent header {#signature-agent-header}

This allows for backward compatibility with existing header agent filtering, and an upgrade to cryptographically secured protocol.
See {{signature-agent}} for more details.

# Security Considerations

## Use of TLS

We reassess {{Section 7.1.2 of HTTP-MESSAGE-SIGNATURES}}.
Clients SHOULD use TLS {{RFC8446}}
(https) or equivalent transport security when making requests with
Message signatures. Failing to do so exposes the Message signature to numerous
attacks that could give attackers unintended access.

This include reverse proxy and their consideration presented in {{reverse-proxy}}.

An origin SHOULD refuse Signature headers when communicated over an unsecured channel.

## Performance Impact

Origins should account for the overhead of signature verification in their operations. A local cache of public keys reduces network requests and verification latency. The choice of signing algorithm impacts CPU requirements. Origins should monitor verification latency and set appropriate timeouts to maintain service levels under load.

## Nonce validation

Clients are the one controlling the nonce. While {{anti-replay}} mandates that clients MUST provide a globally unique nonce, it is the origin's responsibility to enforce it.

Different validation policies have different performance and operational considerations. Global uniqueness requires a global nonce store. Some origins may find that their use case can tolerate sharding on location, timing, or other properties.

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

## Reverse proxy consideration {#reverse-proxy}

An origin may be placed behind a reverse proxy. This means that the proxy is seeing the signature before the origin.

It implies that the proxy sees the Signature before the origin does, may strip it, or even attempt to replay it against other reverse proxies used by the origin.

Origins may require a specific nonce policy to prevent such malicious behaviour and decide to validate the signature themselves.

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

# Test Vectors

## RSASSA-PSS Using SHA-512

The test vectors in this section use the RSA-PSS key defined in {{Appendix B.1.2 of HTTP-MESSAGE-SIGNATURES}}.
This section include non-normative test vectors that may be used as test cases to validate implementation correctness.

### Signature-Agent absent from the request

This example presents a minimal signature using the rsa-pss-sha512 algorithm over test-request. The request does not contain
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"@signature-params": ("@authority")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="yT+sZR1glKOTemVLbmPDFwPScbB1Zj/sMNPEFZcjwJW5jK/taa7HviOXovVwiZOfrrLHS2SbLFUQBxPYZChf7g=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig1`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Input: sig1=("@authority")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="yT+sZR1glKOTemVLbmPDFwPScbB1Zj/sMNPEFZcjwJW5jK/taa7HviOXovVwiZOfrrLHS2SbLFUQBxPYZChf7g=="\
 ;tag="web-bot-auth"
Signature: sig1=:ppXhcGjVp7xaoHGIa7V+hsSxuRgFt8i04K4FWz9ORJtn57t8duD3cyavsnh9grdWWOJHER8ITNBaqe4mKmPq193S+7hSW31IzXSH4/9WfsdrjUBwyJ0fhBU7oNn3UGDqwdhr5TMgVI2/EX8saV5GrOunM09zMEA+d4QWYyKRFJmg+asCs253l2IYPpVp4N55H0uRK7qhb7acng8LNiEPTQZD2s+Kha95LgeciKQSO0jgR/h59fX/dXqLdFIvRMn8Ggs2VUzF/f/MMEXH83gufVnh4SYl/rKMSKWDBsK+OiLpobAVIuIz+HLCVlMnxlkXkhCW2J/Pmo8jht9N5k/y1A==:
~~~

### Signature-Agent included present on the request

This example presents a minimal signature using the rsa-pss-sha512 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="XSHtZVCThSIAksXsH9WBs6AtxtXC0eQGiIcUGSoJstFs8lAWakjhrfwzLhyjtme5iXMZvmFWqDEs6cT3Jf+BbQ=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: "https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="XSHtZVCThSIAksXsH9WBs6AtxtXC0eQGiIcUGSoJstFs8lAWakjhrfwzLhyjtme5iXMZvmFWqDEs6cT3Jf+BbQ=="\
 ;tag="web-bot-auth"
Signature: sig2=:I1QWNzGXdP1a4dSvOHLCVOOanEYHDk+ZsVxM9MLX/p4ko69ghKwR5EOtAD96g7g4GWP7lmpM/jFAf9q8EFRDTPLjUXySwMv4YPgabv2LQihTJG2y8a2m6IGltyruwQNiqSJVUuRaG9+b17CGmAMFZh30X6GXLdQJrCARpeTqPwp2DC+a8haDE/VE5EruqzjA5/2mKwvrkzkSqeW5tOVtFwWRRHIOidquf/8Je6kM9mhgkg4arudLA5SL4wyyYE1jURIgcOl8agrfdJ5Def23DIRtiOLRa8jT9cpTLFAuFHN+mrZA/LH9h0gSIg1cPb+0cMASee5uku1KjWcFer7jWA==:
~~~

## EdDSA Using Curve edwards25519

The test vectors in this section use the Ed25519 key defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.
This section include non-normative test vectors that may be used as test cases to validate implementation correctness.

### Signature-Agent absent from the request

This example presents a minimal signature using the ed25519 algorithm over test-request. The request does not contain
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"@signature-params": ("@authority")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="mYotfW3CUjI68sbGw6oKd7kyXqPjZEtU8xFPGWFrqOAf5qC6MDe3pys3SWWCudB0MvwslHy32WXUpkR7u0lt/w=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig1`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Input: sig1=("@authority")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="mYotfW3CUjI68sbGw6oKd7kyXqPjZEtU8xFPGWFrqOAf5qC6MDe3pys3SWWCudB0MvwslHy32WXUpkR7u0lt/w=="\
 ;tag="web-bot-auth"
Signature: sig1=:+NA/cssf4Y2bQTMTkyvTGRCaVzp9quyUevdwwMtMOWhhOOZ2T1subBj0BtvdnrpDEuwSAbiTeElXDzHL3WWKCw==:
~~~

### Signature-Agent included present on the request

This example presents a minimal signature using the ed25519 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="e8N7S2MFd/qrd6T2R3tdfAuuANngKI7LFtKYI/vowzk4lAZYadIX6wW25MwG7DCT9RUKAJ0qVkU0mEeLElW1qg=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: "https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="e8N7S2MFd/qrd6T2R3tdfAuuANngKI7LFtKYI/vowzk4lAZYadIX6wW25MwG7DCT9RUKAJ0qVkU0mEeLElW1qg=="\
 ;tag="web-bot-auth"
Signature: sig2=:jdq0SqOwHdyHr9+r5jw3iYZH6aNGKijYp/EstF4RQTQdi5N5YYKrD+mCT1HA1nZDsi6nJKuHxUi/5Syp3rLWBA==:
~~~

# Implementations

This draft has a couple of implementations. A demonstration server has been deployed to [https://http-message-signatures-example.research.cloudflare.com/](https://http-message-signatures-example.research.cloudflare.com/).

It uses ed25519 example signing and verifying keys defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.

## Clients

* [Chrome MV3](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Guzzle middleware](https://github.com/olipayne/guzzle-web-bot-auth-middleware) (PHP)

* [Python script](https://zenn.dev/oymk/articles/944069e5eddc27)

* [Linzer](https://github.com/nomadium/linzer/blob/master/spec/integration/cloudflare_example_research_spec.rb) (Ruby)

* [Rust binaries](https://github.com/cloudflare/web-bot-auth) (Rust)

## Servers

* [Caddy plugin](https://github.com/cloudflare/web-bot-auth) (Go)

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

## Test vectors

* In [JSON format](https://github.com/cloudflare/web-bot-auth/blob/main/packages/web-bot-auth/test/test_data/web_bot_auth_architecture_v1.json)


# Acknowledgments
{:numbered="false"}

The editor would also like to thank the following individuals (listed in alphabetical order) for feedback, insight, and implementation of this document -
Marwan Fayed,
Maxime Guerreiro,
Scott Hendrickson,
Jonathan Hoyland,
Mark Nottingham,
Eugenio Panero,
Lucas Pardue,
Malte Ubl,
Loganaden Velvindron,
Tanya Verma.

# Changelog
{:numbered="false"}

v02

- Add response status code
- Add a few reference to improve readability
- Add consideration to sign additional headers
- Add use of TLS in security considerations
- Add RSASSA-PSS example
- Update acknowledgement section
- Reference new implementations: PHP, Python, Ruby, Rust
- Fix signature-agent in the arch diagram to use structured fields
- Fix test vectors to use signature-agent with structured fields
- Fix some typos

v01

- Add mandatory signing of Signature-Agent by clients if present
- Add test vectors for request with and without Signature-Agent
- Update example diagram to be correct
- Add security consideration about reverse proxy
- Update why origin may request a new signature
- Update nonce validation wording and global uniqueness
- Acknowledgements

v00

- Initial draft
- How to leverage HTTP Message Signature to sign request
- How to verify these Signature
- Define web-bot-auth tag to scope this signature
- Derive keyid using JWK Thumbprint
- High level Security and Privacy considerations
