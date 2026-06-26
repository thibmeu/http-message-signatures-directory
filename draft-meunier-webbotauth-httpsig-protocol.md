---
title: "HTTP Message Signatures for automated traffic"
abbrev: "HTTP Message Signatures for Bots"
category: info

docname: draft-meunier-webbotauth-httpsig-protocol-latest
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
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-webbotauth-httpsig-protocol.html"

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
  DIRECTORY: I-D.draft-meunier-http-message-signatures-directory
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  HTTP-MORE-STATUS-CODE: RFC6585
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638


informative:
  CIMD: I-D.draft-ietf-oauth-client-id-metadata-document
  HTTP-BEST-PRACTICES: RFC9205
  OAUTH-BEARER: RFC6750
  RFC8446:
  OWASP-SSRF:
    title: OWASP Server-Side Request Forgery Prevention Cheat Sheet
    target: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
  SIGNATURE-KEY: I-D.draft-hardt-httpbis-signature-key

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
It proposes that every request from bots be signed by a private key owned by its provider.
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
    them like a Bearer from {{OAUTH-BEARER}}. This is impractical to scale for any
    agent beyond select partnerships, and insecure, as key rotation is challenging
    and becomes less secure as the consumers scale.

Using well-established cryptography, we can instead define a simple and secure
mechanism that empowers small and large agents to share their identity.

## HTTP layer choice

This architecture operates solely at the HTTP layer.
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
along with the `Signature-Agent` header for discovery of its verification key.
Upon receiving the request, the Origin ensures it has the verification key for the Agent,
validates the signature, and processes the request if the signature is valid.

## Deployment Models

Signature verification can be performed either directly by origins or delegated
to a fronting proxy. Direct verification by origins provides simplicity and
control. Proxy verification offloads processing and enables shared caching across
multiple origins. The choice depends on traffic volume and operational
requirements.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURES}} defines components to be signed.

Agents MUST include at least one of the following components:

`@authority`
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}

`@target-uri`
: as defined in {{Section 2.2.2 of HTTP-MESSAGE-SIGNATURES}}

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

It is RECOMMENDED the expiry to be no more than 24 hours.

### Signature-Agent {#signature-agent}

`Signature-Agent` is an HTTP Method context header defined in Section 4.1 of {{DIRECTORY}}.
It is RECOMMENDED that the Agent sends requests with `Signature-Agent` header, as described in {{sending-request}}.
If the header is to be sent, one of its members MUST be signed as a component as defined in {{Section 2.1 of HTTP-MESSAGE-SIGNATURES}}.
The `Signature-Agent` member identifies where candidate key material can be found.
The key used to verify the signature is selected by the `keyid` parameter of the
corresponding `Signature-Input` member.

This results in the following components to be signed

~~~
("@authority" "signature-agent";key="sig1")
~~~

It is RECOMMENDED that the `key` matches the signature label.

### Multiple signatures {#multiple-signatures}

A request MAY contain more than one Web Bot Auth signature. Each signature is
identified by its HTTP Message Signatures label. When `Signature-Agent` is
present, each signer SHOULD provide a `Signature-Agent` member for its label.

A signer MAY cover selected members from another signature label, including
`"signature";key=...`, `"signature-input";key=...`, and
`"signature-agent";key=...`. This lets a signer preserve evidence that another
signer contributed to the request.

This records which parties signed which fields. It does not, by itself, express
authorization, delegation, or consent. Those meanings are deployment policy or
are carried in separately signed fields. The delegation examples in
{{DIRECTORY}} cover a different problem: how key material can show delegation.
Multiple signatures only show which signatures were present and what they
covered.

### Anti-replay {#anti-replay}

Origins MAY want to prevent signatures from being spoofed or used multiple times by bad actors and thus require a `nonce` to be added to the `@signature-params`.
This is described in {{Section 7.2.2 of HTTP-MESSAGE-SIGNATURES}}.

Agents SHOULD extend `@signature-parameters` defined in {{generating-http-message-signature}} as follows:

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
+---------+                                                                                  +----------+
|         |                                     Exchange                                     |          |
|         |<================================  Cryptographic  ===============================>|          |
|         |                                     material                                     |          |
|  Agent  |                                                                                  |  Origin  |
|         |     .-------------------------------------------------------------------.        |          |
|         +-----| GET /path/to/resource                                             |------->|          |
|         |     | Signature: sig=abc==                                              |        |          |
+---------+     | Signature-Input: sig=("@authority" "signature-agent";key="sig");\ |        +----------+
                |                  created=1700000000;\                             |
                |                  expires=1700011111;\                             |
                |                  keyid="ba3e64==";\                               |
                |                  tag="web-bot-auth"                               |
                | Signature-Agent: sig="https://signer.example.com"                 |
                '-------------------------------------------------------------------'
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
- During step 5, if the keyid is unknown to the origin, they MAY fetch key material as indicated by the `Signature-Agent` header defined in Section 4 of {{DIRECTORY}}.

Origin MAY require the `nonce` to satisfy certain constraints: be globally unique using a global nonce store, be unique to a specific location or time window using a local cache, or no constraint at all.

## Key Distribution and Discovery {#key-distribution-and-discovery}

This section describes key discovery for the agent.

The reference for discovery is a URL. It SHOULD be an HTTPS URL. The
`Signature-Agent` header defines typed discovery. This architecture uses these
types:

`directory`
: Resolve the HTTP Message Signatures Directory at the well-known URI registered
in {{DIRECTORY}}. This is the default when no `type` parameter is present.

`jwks_uri`
: Resolve the member value as a direct JWK Set URI.

`cimd`
: Resolve the member value as a Client ID Metadata Document {{CIMD}} URI. The
document then provides key material through `jwks` or `jwks_uri`.

For all types, the key is selected using the `keyid` parameter in
`Signature-Input`.

Examples:

~~~
Signature-Agent: sig1="https://signer.example.com"
Signature-Agent: sig2="https://signer.example.com/.well-known/jwks.json";type=jwks_uri
Signature-Agent: sig3="https://signer.example.com/mybot";type=cimd
~~~

Example:

~~~json
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


### Out-of-band communication between client and origin

A service submitting their key to an origin, or the origin manually adding a service to their trusted list.

### Public list

Could be a GitHub repository like the public suffix list. The issue is the gating of such repositories, and therefore its governance.

### Signature-Agent header {#signature-agent-header}

This allows for backward compatibility with existing header agent filtering, and an upgrade to a cryptographically secured protocol.
See {{signature-agent}} for more details.

### Signature-Key header

{{SIGNATURE-KEY}} defines a separate key discovery header for HTTP Message
Signatures. Deployments MAY use it when they need that model. This architecture
uses `Signature-Agent` as its default discovery mechanism.

## Session Protocol Considerations

Per-request signature generation and verification may incur computational overhead from cryptographic operations and key discovery. For high-frequency interactions, origins might establish sessions to reduce repeated verification.

One approach: after successful signature verification, an origin issues a session credential (e.g., an HTTP cookie) that subsequent requests present in lieu of a full signature. This trades cryptographic verification costs for the security properties of bearer tokens, including susceptibility to credential theft and replay within the session lifetime.

The design of session protocols, including appropriate session lifetimes and binding mechanisms, is out of scope for this document.

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

## Nonce validation {#nonce-validation}

Clients control the nonce. While {{anti-replay}} mandates that clients MUST provide a globally unique nonce, it is the origin's responsibility to enforce it.

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

Implementations SHOULD NOT reuse a signing key for different purposes. For
example, if an agent implementor has two agents they want to differentiate,
these should use distinct signing keys and signing key directories.

## Reverse proxy consideration {#reverse-proxy}

An origin may be placed behind a reverse proxy, which means the proxy will see
the `Signature` and `Signature-Agent` headers before the origin does.
A proxy SHOULD NOT strip the `Signature` or `Signature-Agent` headers from
requests.

A proxy SHOULD NOT replay signatures against other reverse proxies used by the
origin, as this allows impersonation of the principal signature agent.

Origins MAY require a specific nonce policy to prevent such malicious behaviour
and decide to validate the signature themselves. This has to be done in
accordance with {{nonce-validation}}. For example, an origin could
require a nonce derived from public information (such as the current date),
mandate nonce chaining (where each nonce is the hash of the previous one),
or provide its own nonce in an `Accept-Signature` response to challenge the agent.

Such policies MAY incur additional round-trip between the client and the origin
to convey `accept-signature` header, or deployment specific exchanges.

### Signature-Agent labeling

An intermediary is allowed to relabel an existing signature when processing the
message, per {{Section 7.2.5 of HTTP-MESSAGE-SIGNATURES}}.

This MAY apply to `Signature-Agent`, when included in the request as defined in
{{signature-agent}},
An intermediary updating the member key MUST update the components of the
associated signatures accordingly.

For instance, an intermediary updating the `Signature-Agent` from `sig2` to
`sig3` on the example provided in {{example-signature-agent-included}} would
result in the following `Signature`, `Signature-Input`, and `Signature-Agent`
header.

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: sig3="https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent";key="sig3")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="XSHtZVCThSIAksXsH9WBs6AtxtXC0eQGiIcUGSoJstFs8lAWakjhrfwzLhyjtme5iXMZvmFWqDEs6cT3Jf+BbQ=="\
 ;tag="web-bot-auth"
Signature: sig2=:I1QWNzGXdP1a4dSvOHLCVOOanEYHDk+ZsVxM9MLX/p4ko69ghKwR5EOtAD96g7g4GWP7lmpM/jFAf9q8EFRDTPLjUXySwMv4YPgabv2LQihTJG2y8a2m6IGltyruwQNiqSJVUuRaG9+b17CGmAMFZh30X6GXLdQJrCARpeTqPwp2DC+a8haDE/VE5EruqzjA5/2mKwvrkzkSqeW5tOVtFwWRRHIOidquf/8Je6kM9mhgkg4arudLA5SL4wyyYE1jURIgcOl8agrfdJ5Def23DIRtiOLRa8jT9cpTLFAuFHN+mrZA/LH9h0gSIg1cPb+0cMASee5uku1KjWcFer7jWA==:
~~~

`Signature` is unchanged as the base is similar. Both `Signature-Agent` and
`Signature-Input` reflect the update from `sig2` to `sig3`.

## Server-Side Request Forgery (SSRF) {#ssrf}

As described in {{key-distribution-and-discovery}}, verifiers may fetch key directories based on
the value conveyed in `Signature-Agent` when included in a request. Since
clients control the `Signature-Agent` header value, this introduces a risk of
server-side request forgery (SSRF) attacks by malicious clients.

Verifiers SHOULD take appropriate precautions as follows:

`Response size`
: a directory can be arbitrarily large. Verifiers SHOULD reject responses
  exceeding a defined byte limit after content decoding.

`Key count`
: a JWKS with many keys forces O(n) key search. Verifiers SHOULD enforce
  a maximum key count.

`Fetch latency`
: no timeout allows slowloris-style exhaustion. Verifiers SHOULD apply
  a wall-clock timeout to directory fetches.

`Redirect chains`
: unbounded HTTP redirects can be used to amplify requests. Verifiers
  SHOULD limit redirect depth.

`Network address ranges`
: no address filtering can target internal services. Verifiers SHOULD
  prevent directory fetches to private, loopback, and link-local
  address ranges.

Further recommendations can be found in the Open Worldwide Application
Security Project (OWASP) SSRF Prevention Cheat Sheet {{OWASP-SSRF}}.

## Test and Demonstration Keys

Test keys, including the example keys in {{HTTP-MESSAGE-SIGNATURES}}, MUST NOT
be used in production. Verifiers SHOULD reject known test keys when they are
detected in key directories or out-of-band configuration.

## Static Signatures

Deployments MUST NOT treat a precomputed Web Bot Auth signature as a long-lived
access credential. A reusable static signature has bearer-token semantics and can
be replayed until the covered signature parameters, key, or verifier policy make
it unusable.

Agents SHOULD generate signatures for the request being sent, with bounded
`created` and `expires` values. Long expiration windows increase replay risk.

## Discovery Failure

Failure to fetch or validate a key directory, beyond the directory cache window
discussed in {{cache-behaviour}}, means that the asserted identity is not
verified. It does not prove that the signer is malicious, and it does not make
the request trusted. The resulting enforcement decision is local policy.

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

# Deployment Guidance

This appendix is operational guidance. It does not define new protocol
requirements.

## Verifier Outcomes

Verifiers should keep three outcomes distinct:

`verified`
: the signature and key material validate.

`invalid`
: the signature, covered components, key, or freshness checks fail.

`unverified`
: the verifier cannot obtain enough information to decide, for example because
  directory discovery failed or the key is unknown.

Origins can apply local policy to each outcome. During deployment, treating
`unverified` as one bot-management signal is safer than treating it as either
`verified` or `invalid`.

## Directory Availability

Directory resources are bootstrap material. Operators serving a directory should
make it reachable without requiring Web Bot Auth on the directory request. They
should also avoid bot protection rules that block ordinary verifier fetches of
the well-known resource.

The directory endpoint should support `GET`. Supporting `HEAD`, `ETag`,
`Last-Modified`, `Cache-Control`, and conditional requests can reduce fetch
load. Cache is specifically discussed in {{cache-behaviour}}.

## Bounded Directory Fetches

Verifiers fetch directories named by untrusted requests. Fetches should be
bounded as described in {{ssrf}}. In particular, verifiers should use a fetch
timeout, bound the decoded response size, bound the number of keys considered,
limit redirects, and prevent fetches to private, loopback, and link-local
addresses.

Verifiers should also coalesce concurrent fetches for the same directory and
apply per-directory or per-origin concurrency limits. This avoids a fetch storm
when many requests reference the same uncached directory.

## Cache Behaviour {#cache-behaviour}

Verifiers should use normal HTTP caching semantics {{HTTP-CACHE}} for key
directories. In particular, verifiers should respect `Cache-Control`, `Expires`,
`Date`, `ETag`, and `Last-Modified` when present.

A verifier should not fetch the directory for every request. It should refresh
cached directories when they become stale, and can use background refresh with
jitter to avoid synchronized refetches.

## Negative Caching and Retry

Verifiers can cache unsuccessful discovery outcomes for a short period to reduce
repeated fetches. Negative cache entries should expire after no more than five
minutes. They are operational throttling state, not proof that a signature is
invalid.

Network failures, TLS failures, and `5xx` responses should be treated as
transient unless local policy says otherwise. Verifiers should retry with bounded
exponential backoff and jitter. When a directory response includes
`Retry-After`, verifiers should respect it as described by {{HTTP}} and
{{HTTP-BEST-PRACTICES}}.

## Freshness and Replay

Shorter signature lifetimes reduce replay risk but increase sensitivity to clock
skew and signing failures. Nonces provide stronger replay defense, but require
state at the verifier. Some deployments can tolerate bounded replay for short
windows; others need strict {{nonce-validation}}.

These choices are deployment policy. Verifiers should avoid accepting signatures
with freshness windows longer than their risk model permits.

## Rollout and Fallback

Web Bot Auth deployments will coexist with existing bot identification signals
during rollout. Verifiers can continue to use existing methods such as IP-based
checks, forward-confirmed reverse DNS, local allowlists, and reputation systems.

Fallback should not turn an unsupported or unverifiable Web Bot Auth signature
into a trusted identity. It should leave the request in the origin's existing
bot-management path.

## Proxies and Intermediaries

Proxies and intermediaries need to preserve the fields covered by a signature if
the origin will verify that signature. If a proxy rewrites the authority, path,
or signed header fields, the origin may no longer see the message that was
signed.

A deployment can instead verify at the proxy and pass the result to the origin
through a deployment-local trusted channel. That assertion is local policy; it is
not a replacement for the original HTTP Message Signature.

## CORS

Key directories contain public key material. If browser-based verifiers need to
fetch them cross-origin, a directory server can use a permissive CORS policy such
as `Access-Control-Allow-Origin: *` without credentials. CORS is not key
authentication and does not replace signature validation.

## Deployment Anti-Patterns

Deployments should avoid:

* using test or demonstration keys in production
* issuing one static signature for many requests
* asking users to copy long-lived signatures into third-party tools
* sharing one signing key across unrelated agents or purposes
* relying on manual key rotation as the only revocation mechanism

# Examples

## Multiple signatures with a remote browser {#example-multiple-signatures}

This example shows Alice's agent using a remote browser to fetch a resource. The
agent signs selected request fields. The remote browser signs the request it
sends to the origin and also covers the agent's signature fields. The signature
values are illustrative; this is not a test vector.

~~~
NOTE: '\' line wrapping per RFC 8792

GET /resource HTTP/1.1
Host: origin.example
Signature-Agent: agent="https://agent.alice.example",\
 browser="https://browser.example"
Signature-Input: agent=("@method" "@authority" "@path"\
 "signature-agent";key="agent");created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;tag="web-bot-auth",\
 browser=("@method" "@authority" "@path"\
 "signature-agent";key="browser"\
 "signature-agent";key="agent"\
 "signature-input";key="agent"\
 "signature";key="agent");created=1735689601\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;tag="web-bot-auth"
Signature: agent=:YWdlbnQtc2lnbmF0dXJl:,\
 browser=:YnJvd3Nlci1zaWduYXR1cmU=:
~~~

The origin can verify both signatures. The `agent` signature covers the fields
selected by Alice's agent. The `browser` signature covers the request sent by the
remote browser, its own `Signature-Agent` member, and the `agent` signature
fields. This records that the remote browser forwarded a request carrying the
agent's signature. It does not say that Alice's agent authorized the remote
browser to act for it.

# Test Vectors

## RSASSA-PSS Using SHA-512

The test vectors in this section use the RSA-PSS key defined in {{Appendix B.1.2 of HTTP-MESSAGE-SIGNATURES}}.
This section includes non-normative test vectors that may be used as test cases to validate implementation correctness.

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
 ;expires=4889289600\
 ;nonce="EfK54mBzFxPqwpmZ430GZRqVGrLT/DplPWuFIM1jLJDjrAIX3yFGidftF1h1+zLHfjoNKhx74yU1psH1XD7BeA=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig1`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Input: sig1=("@authority")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=4889289600\
 ;nonce="EfK54mBzFxPqwpmZ430GZRqVGrLT/DplPWuFIM1jLJDjrAIX3yFGidftF1h1+zLHfjoNKhx74yU1psH1XD7BeA=="\
 ;tag="web-bot-auth"
Signature: sig1=:Bqj+UQfJNSRx0Dz/K/4/+Bo1l8UUH5Ps1zYzX6H6nKCyZJ88Hry/KZF2JishxI1h9+LJTmRmDmw2HxbUeZkoUUgmLbg168GWiYFBK0IQRKQvvbnzrONutKNmanvIXNvrN2ZB2h+w9ekSol3XJRncErrwcU2PWltBR+An4H2kIiRBfnBRi85eCVF+s6SYRxoAJvRo6avTCvCZe9Gvw8Ezbj8QnHU37uvTN72+MBDEsFN94ozfAT8MTB4wAwqXYLMf9mnl0mpK2UbnXrzgffRxOhEHVvHNIN8aB7ThM1p4JzaTN1HuXQFPYOWgCojOCv2IovGOygai/j3p4PzMJUp4Lw==:
~~~

### Signature-Agent included present on the request {#example-signature-agent-included}

This example presents a minimal signature using the rsa-pss-sha512 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent";key="agent2": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent";key="agent2")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=4889289600\
 ;nonce="zJwYV5pG8TA9NnaOu9RBShBXtiWuyoWthZXQBT2J77XTpW3ADk49DlbOvpqjJqy3SH3lyNVS/Zo0DmKQX8HYuQ=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: agent2="https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent";key="agent2")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=4889289600\
 ;nonce="zJwYV5pG8TA9NnaOu9RBShBXtiWuyoWthZXQBT2J77XTpW3ADk49DlbOvpqjJqy3SH3lyNVS/Zo0DmKQX8HYuQ=="\
 ;tag="web-bot-auth"
Signature: sig2=:ngb8Yuk2zY/O5nyApob/uwIRWNE1md5xrzYSpPfVCWMHMjdQhj8HTPY8lrE8jHDHRtpqUy7jvYM8LzaHb1NGyxPemVMEOoZpBWXxboqSbp1LTAb2o5qbETmSuDM7UZE4WuSDQoIG5GF5AZ8b8lFEWDP1pw0XV1zsZMn8EPU/DbTkFtGgVPdGehjywJRqnXCXEX0wRCGg4+nTJwWs736JqgbBCuafQPCdwITQucMyGA12QOmMc8eQUdjcS/uqzkDxj1+iI3PDCYnscUTHcGuNv6rWxIx0D+rqWhOoLeYwzDPUm3qs2utVCATIgK0ktLWSfGcPK6p3IwJIUj7cSkbVRg==:
~~~

### Legacy Signature-Agent (sf-string instead of sf-dictionary) included present on the request

THIS IS A LEGACY EXAMPLE. IF YOU ARE AN IMPLEMENTER, PLEASE UPDATE TO THE ABOVE.

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
 ;expires=4889289600\
 ;nonce="g0iqFa9e1ffijlyOScDkXpfSmTbYpRNSGPJrQ1It20ahwgzB3jOUcdgLgFxUg7RMtW4V8IILaKKtA+YuSyIgJQ=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig1`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Input: sig1=("@authority")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=4889289600\
 ;nonce="g0iqFa9e1ffijlyOScDkXpfSmTbYpRNSGPJrQ1It20ahwgzB3jOUcdgLgFxUg7RMtW4V8IILaKKtA+YuSyIgJQ=="\
 ;tag="web-bot-auth"
Signature: sig1=:FFASViSdcgsyaqqYiCnkHreeZzbNKcTzDvZC5uVlP/dn9IbWj8j0o4wKFTH3rBnUiSUBduwm1Gp5VlIPCp01Ag==:
~~~

### Signature-Agent included present on the request

This example presents a minimal signature using the ed25519 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent";key="agent2": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent";key="agent2")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=4889289600\
 ;nonce="XeP72svPKNiGEg3aDE7WJuTpN69H08oMFqC8NLFy1MptpENAT3WZTYwK+MYdsFMlaqHCJGo9ZAhqer1NWY9Epg=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: agent2="https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent";key="agent2")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=4889289600\
 ;nonce="XeP72svPKNiGEg3aDE7WJuTpN69H08oMFqC8NLFy1MptpENAT3WZTYwK+MYdsFMlaqHCJGo9ZAhqer1NWY9Epg=="\
 ;tag="web-bot-auth"
Signature: sig2=:DGiW2ErlQh0hc8wY2FQdbnFd6CEmonyY8nlvECIJFaUSYYNvNvSsGyP99BUGtq51gA4ouXlkUwjnta084bpjCg==:
~~~

### Legacy Signature-Agent (sf-string instead of sf-dictionary) included present on the request

THIS IS A LEGACY EXAMPLE. IF YOU ARE AN IMPLEMENTER, PLEASE UPDATE TO THE ABOVE.

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

This draft has a couple of public implementations. A demonstration server has been deployed to [https://http-message-signatures-example.research.cloudflare.com/](https://http-message-signatures-example.research.cloudflare.com/).

It uses ed25519 example signing and verifying keys defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.

## Clients

draft-meunier-webbotauth-httpsig-protocol-00

* [Chrome MV3](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Rust binaries](https://github.com/cloudflare/web-bot-auth) (Rust)

draft-meunier-web-bot-auth-architecture-03

* [Puppeteer script](https://github.com/stytchauth/web-bot-auth-example) (JavaScript)

* [Guzzle middleware](https://github.com/olipayne/guzzle-web-bot-auth-middleware) (PHP)

* [Python script](https://zenn.dev/oymk/articles/944069e5eddc27) (Python)

* [Bot-Authentication](https://github.com/cyberstormdotmu/bot-authentication) (Python)

* [HTTPie plugin](https://github.com/cloudflare/web-bot-auth) (Python)

* [Web scrapers (scrapy/crawl4ai)](https://github.com/cyberstormdotmu/bot-authentication) (Python)

* [HUMAN Verified AI Agents](https://github.com/HumanSecurity/human-verified-ai-agent) (Python)

* [Linzer](https://github.com/nomadium/linzer/blob/master/spec/integration/cloudflare_example_research_spec.rb) (Ruby)

## Servers

draft-meunier-webbotauth-httpsig-protocol-00

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

draft-meunier-web-bot-auth-architecture-03

* [Caddy plugin](https://github.com/cloudflare/web-bot-auth) (Go)

* [Apache module](https://github.com/garyillyes/web-bot-auth-apache) (C)

## Test vectors

* In [JSON format](https://github.com/cloudflare/web-bot-auth/blob/main/packages/web-bot-auth/test/test_data/web_bot_auth_architecture_v2.json)


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

draft-meunier-webbotauth-httpsig-protocol-00

- Rename draft from `draft-meunier-web-bot-auth-architecture`.
- Add SSRF guidance for `Signature-Agent` directory fetches.
- Add deployment guidance for verifier outcomes, directory fetches, caching,
  retry, rollout, proxies, CORS, and observability.
- Add guidance for test keys, static signatures, and discovery failures.
- Add multiple Web Bot Auth signatures and an example.
- Add typed `Signature-Agent` discovery examples for `directory`, `jwks_uri`,
  and `cimd`.
- Group implementations by the draft version that added them.
- Clarify that `Signature-Input` `keyid` selects the key and `Signature-Agent`
  points to candidate key material.
- Note `Signature-Key` as an optional discovery header.
- Fix typos.

draft-meunier-web-bot-auth-architecture-05

- Add Sandor Major as an author.
- Add session protocol considerations.
- Update HTTP Message Signatures test vectors.
- Keep legacy `Signature-Agent` string examples for implementers migrating to
  dictionary members.

draft-meunier-web-bot-auth-architecture-04

- Change `Signature-Agent` to a Structured Fields dictionary.
- Add a security consideration for intermediaries that relabel
  `Signature-Agent` members.
- Allow `@target-uri` as a replacement for `@authority`.
- Add contributors.
- Add implementations.
- Remove the `purpose` field from the Web Bot Auth example.

draft-meunier-web-bot-auth-architecture-03

- Update the Linzer example URL.
- Fix the section reference and name for status code 429.
- Fix typos.

draft-meunier-web-bot-auth-architecture-02

- Add response status codes.
- Add references for readability.
- Add text about signing extra headers.
- Add TLS guidance to Security Considerations.
- Add RSASSA-PSS examples.
- Update acknowledgments.
- Add PHP, Python, Ruby, and Rust implementations.
- Fix `Signature-Agent` in the architecture diagram to use Structured Fields.
- Fix test vectors to use Structured Fields for `Signature-Agent`.
- Fix typos.

draft-meunier-web-bot-auth-architecture-01

- Require clients to sign `Signature-Agent` when it is present.
- Add test vectors for requests with and without `Signature-Agent`.
- Fix the example diagram.
- Add reverse proxy security considerations.
- Update text about why an origin may request a new signature.
- Update nonce validation wording and uniqueness requirements.
- Add acknowledgments.

draft-meunier-web-bot-auth-architecture-00

- Initial draft.
- Describe how to use HTTP Message Signatures to sign requests.
- Describe signature verification.
- Define the `web-bot-auth` tag.
- Derive `keyid` from the JWK Thumbprint.
- Add initial Security and Privacy Considerations.
