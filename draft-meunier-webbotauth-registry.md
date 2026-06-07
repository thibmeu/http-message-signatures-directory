---
title: "Registry and Signature Agent card for Web bot auth"
abbrev: "Registry and Signature Agent card for Web bot auth"
category: info

docname: draft-meunier-webbotauth-registry-latest
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
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-webbotauth-registry.html"

author:
 -
    fullname: Maxime Guerreiro
    organization: Cloudflare
    email: maxime.guerreiro@gmail.com
 -
    fullname: Ulas Kirazci
    organization: Amazon
    email: ulaskira@amazon.com
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  ABNF: RFC5234
  AIPREF-VOCAB: I-D.draft-ietf-aipref-vocab
  CDDL: RFC8610
  CIMD: I-D.draft-ietf-oauth-client-id-metadata-document
  DCR: RFC7591
  DIRECTORY: I-D.draft-meunier-http-message-signatures-directory
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  HTTP-MORE-STATUS-CODE: RFC6585
  JAFAR: I-D.draft-illyes-webbotauth-jafar
  JWK: RFC7517
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638
  WEB-LINKING: RFC8288


informative:
  CBCP: I-D.draft-illyes-webbotauth-cbcp
  DATAURL: RFC2397
  OAUTH-BEARER: RFC6750
  OPENID-CONNECT-DISCOVERY:
    title: OpenID Connect Discovery 1.0
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  PSK-TLS: RFC9257
  RATELIMIT-HEADER: I-D.draft-ietf-httpapi-ratelimit-headers
  RFC8446:
  ROBOTSTXT: RFC9309
  SIGNATURE-KEY: I-D.draft-hardt-httpbis-signature-key
  UTF8: RFC3629

--- abstract

This document defines the "Signature Agent Card", a JSON metadata document that a
signature agent using {{DIRECTORY}} publishes to describe itself: its identity,
purpose, rate expectations, and cryptographic keys. Its parameters are drawn from
the OAuth Dynamic Client Registration Metadata registry {{DCR}}, the same
namespace used by {{CIMD}}, extended with a single `web_bot_auth` object. This
document registers that object with IANA and establishes a registry for its
members.

--- middle

# Introduction

Signature Agents are entities that originate or forward signed HTTP requests on behalf
of users or services. They include bots developers, platforms providers,
and other intermediaries using {{DIRECTORY}}. These agents often
need to identify themselves, and establish
trust with origin servers.

Today, the mechanisms for doing so are inconsistent: some rely on User-Agent
strings (e.g. `MyCompanyBot/1.0`), others on IP address lists hosted on file servers (e.g. `https://curated-bots.com/ips.json`), and still others on out-of-band
definitions (e.g. documentation on docs.example.com/mybot). This diversity makes it difficult for operators and origin servers
to reliably discover and share a Signature Agent’s purpose, contact information, or rate
expectations.

The OAuth Client ID Metadata Document {{CIMD}} already defines a resolvable
identity: a `client_id` that is an HTTPS URL dereferencing to a JSON metadata
document. This document reuses it.
A Signature Agent Card takes the client metadata parameters that {{CIMD}} draws
from {{DCR}}, and adds bot-specific information in a single `web_bot_auth`
object. An existing Client ID Metadata Document is therefore a valid Signature
Agent Card.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Signature Agent Card {#signature-agent-card}

A Signature Agent Card is a JSON object describing a Signature Agent. Its
parameters are drawn from the OAuth Dynamic Client Registration Metadata registry
{{DCR}}, the same parameter namespace used by {{CIMD}}. Web bot auth-specific
information is carried in a single `web_bot_auth` object,
defined in {{web-bot-auth-extension}}.

Because the card reuses the {{CIMD}} parameter namespace, an existing Client ID
Metadata Document is a valid Signature Agent Card. A client that does not
understand the `web_bot_auth` object ignores it.

The Signature-Agent header is defined in {{Section 4.1 of DIRECTORY}} and may
reference a Signature Agent Card, as described in {{cimd-discovery}}.

~~~
{
  "client_id": "https://example.com/bot",
  "client_name": "Example Bot",
  "client_uri": "https://example.com/bot/about.html",
  "logo_uri": "https://example.com/logo.png",
  "contacts": ["mailto:bot-support@example.com"],
  "jwks_uri": "https://example.com/.well-known/http-message-signatures-directory",
  "web_bot_auth": {
    "expected-user-agent": "Mozilla/5.0 ExampleBot",
    "rfc9309-product-token": "ExampleBot",
    "rfc9309-compliance": ["User-Agent", "Allow", "Disallow", "Content-Usage"],
    "trigger": "fetcher",
    "purpose": "tdm",
    "targeted-content": "Cat pictures",
    "rate-control": "429",
    "rate-expectation": "avg=10rps;max=100rps",
    "known-urls": ["/", "/robots.txt", "*.png"]
  }
}
~~~

Unless otherwise specified, all parameters in this document are OPTIONAL, except
that a card resolved through its `client_id` MUST carry `client_id` (see
{{signature-agent-parameter-client-id}}). There MUST be at least one parameter
set.

Parameters for which the value is unknown MUST be ignored.
All string values are UTF-8.

## Client Metadata Parameters {#client-metadata-parameters}

The following parameters are OAuth client metadata parameters defined in {{DCR}}
and used as in {{CIMD}}. They keep the semantics defined there; this section only
notes how they are used by a Signature Agent.

`client_id`
: A resolvable identifier for the Signature Agent. See
{{signature-agent-parameter-client-id}}.

`client_name`
: A friendly name for the Signature Agent, e.g. `ExampleBot`.

`client_uri`
: A URL of a web page describing the Signature Agent: what it does, and how it
handles the data it fetches, e.g. `https://example.com/bot/about.html`.

`logo_uri`
: A URL referencing a logo for the Signature Agent, for quick visual
identification, e.g. `https://example.com/logo.png`.

`contacts`
: An array of URIs providing reliable communication channels, typically email
addresses, e.g. `["mailto:bot-support@example.com"]`.

`jwks_uri`
: See {{signature-agent-parameter-jwks-uri}}.

`jwks`
: A JWK Set as defined in {{Section 5 of JWK}}. See
{{signature-agent-parameter-jwks}}.

### Client ID {#signature-agent-parameter-client-id}

The `client_id` parameter is the resolvable identity of the Signature Agent, as
defined in {{CIMD}}. It is an HTTPS URL that, when dereferenced, returns the
Signature Agent Card. A card retrieved through its `client_id` MUST include the
`client_id` parameter.

A client retrieving a card from a URL MUST use the `GET` method, MUST treat any
status code other than `200 (OK)` as an error, and MUST NOT follow HTTP redirects.
The `client_id` value in the returned document MUST be identical to the URL used
to retrieve it, compared as described in {{Section 6.2.1 of !RFC3986}} (simple
string comparison).

Example

* `https://example.com/bot`

### JWKS URI {#signature-agent-parameter-jwks-uri}

The `jwks_uri` parameter provides the URL of a JWK Set as defined in
{{Section 5 of JWK}}, following the {{DCR}} semantics. The HTTP Message Signatures
Directory defined in {{DIRECTORY}} is such a JWK Set, so a consumer that
dereferences `jwks_uri` reads the agent's keys whether or not it understands the
directory media type. {{DCR}} does not mandate a media type for the resource at
`jwks_uri`.

When present, this parameter separates key material discovery from metadata
discovery. Clients that need key material SHOULD fetch the directory at the
given URL rather than relying on the `jwks` parameter. This separation allows
registry operators to host metadata and key material on different endpoints,
supporting deployment scenarios where the registry endpoint itself contains
signature agent card metadata but the key directory is hosted elsewhere.

If both `jwks_uri` and `jwks` are present, the
`jwks_uri` takes precedence for key discovery.

The resource at `jwks_uri` SHOULD be signed using {{HTTP-MESSAGE-SIGNATURES}}, as
a directory is in {{DIRECTORY}}. Clients SHOULD validate the signature and ignore
keys that do not carry a corresponding valid signature.

The URI scheme MUST be `https`.

Example

* `https://example.com/.well-known/http-message-signatures-directory`

### JWKS {#signature-agent-parameter-jwks}

The `jwks` parameter contains a JWK Set as defined in {{Section 5 of JWK}}. It is
the inline client metadata parameter defined in {{DCR}}.

If `jwks` is present, it is RECOMMENDED that the card is signed using {{HTTP-MESSAGE-SIGNATURES}}.
`Content-Digest` header MUST be included in the covered components.

TODO: describe signature, CWS keys.

## Web Bot Auth Extension {#web-bot-auth-extension}

The `web_bot_auth` parameter is a JSON object carrying bot-specific metadata. It
is registered in the OAuth Dynamic Client Registration Metadata registry {{DCR}}
(see {{iana}}), so that it may appear in any Client ID Metadata Document {{CIMD}}.
Its members are governed by the "Web Bot Auth Metadata Parameters" registry
defined in {{iana}}. Members for which the value is unknown MUST be ignored.

The members defined by this document are below.

### Expected user agent {#signature-agent-parameter-user-agent}

The `expected-user-agent` parameter specifies one or more `User-Agent` strings as defined in {{Section 10.1.5 of HTTP}}
or prefix matches. Prefixes MAY use `*` as a wildcard.

Example

* `Mozilla/5.0 ExampleBot`

### robots.txt product token {#signature-agent-parameter-robotstxt-token}

The `rfc9309-product-token` parameter specifies the product token used for
`robots.txt` directives per {{Section 2.2.1 of ROBOTSTXT}}.

Example

* `ExampleBot`

### robots.txt compliance {#signature-agent-parameter-robotstxt-compliance}

The `rfc9309-compliance` parameter lists directives from `robots.txt` that the
agent implements.

Example

* `["User-Agent", "Disallow"]`
* `["User-Agent", "Disallow", "CrawlDelay"]`

### Trigger {#signature-agent-parameter-trigger}

The `trigger` parameter indicates the operational mode of the agent.

Valid values:

1. `fetcher` - request initiated by the user
2. `crawler` - autonomous scanning

### Purpose {#signature-agent-parameter-purpose}

The `purpose` parameter describes the intended use of collected data. Values
SHOULD be drawn from a controlled vocabulary, such as {{AIPREF-VOCAB}}.

Example

* `search`
* `tdm`

### Targeted content {#signature-agent-parameter-targeted-content}

The `targeted-content` parameter specifies the type of data the agent seeks.
Its format is arbitrary UTF-8 encoded string.

Example

* `SEO analysis`
* `Vulnerability scanning`
* `Ads verification`

### Rate control {#signature-agent-parameter-rate-control}

The `rate-control` parameter indicates how origins can influence the agent’s
request rate.

TODO: specify a format

Example

* CrawlDelay in robots.txt (non-standard)
* Custom tool
* 429 + {{RATELIMIT-HEADER}}

### Rate expectation {#signature-agent-parameter-rate-expectation}

The `rate-expectation` parameter specifies anticipated request volume or
burstiness.

TODO: consider a format such as `avg=10rps;max=100rps`

Example

* 500 rps
* Spikes during reindexing

### Known URLs {#signature-agent-parameter-known-urls}

The `known-urls` parameter lists predictable endpoints accessed by the agent.

These URLs may be absolute URLs like `https://example.com/index.html`.
They could be relative path like `/ads.txt`.
Or they can use `*` as wildcard such as `*.png`.

Example

* `["/"]`
* `["/ads.txt"]`
* `["/favicon.ico"]`
* `["/index.html"]`

# Discovery

A Signature Agent Card is discovered by dereferencing its `client_id`, as
described in {{cimd-discovery}}, or from a registry.

A registry is a list of URLs, each refering to a signature agent card. An entry
MAY be the `client_id` of a Signature Agent Card.

The URI scheme MUST be one of:

* https: Points to an HTTPS resource serving a signature agent card
* data: Contains an inline signature agent card

## CIMD-based discovery {#cimd-discovery}

A Signature Agent advertises its identity through a resolvable `client_id` URL,
as defined in {{CIMD}}. A consumer discovers the agent's metadata and keys as
follows:

1. Obtain a `client_id` URL, from a `Signature-Agent` header
   ({{signature-agent-header}}), a registry entry, or out of band.
2. Dereference the `client_id` with a `GET` request. The response MUST be
   `200 (OK)`, redirects MUST NOT be followed, and the returned `client_id` MUST
   match the requested URL (see {{signature-agent-parameter-client-id}}). The
   response is the Signature Agent Card.
3. Obtain key material. If the card has a `jwks_uri`, fetch the HTTP Message
   Signatures Directory at that URL as defined in {{DIRECTORY}}; otherwise use the
   inline `jwks` parameter.

The metadata document and the key directory are distinct resources reached at
different hops, so no content negotiation between them is required: the directory
is served with its own media type as defined in {{DIRECTORY}}, while the metadata
document carries no mandated media type.

~~~
GET /bot HTTP/1.1
Host: example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_id": "https://example.com/bot",
  "client_name": "Example Bot",
  "jwks_uri": "https://example.com/.well-known/http-message-signatures-directory",
  "web_bot_auth": {
    "trigger": "fetcher",
    "purpose": "tdm"
  }
}
~~~

Example

~~~txt
# An example list of bots
https://bot1.example.com/.well-known/http-message-signatures-directory
https://crawler2.example.com/.well-known/http-message-signatures-directory

# Now the list of platforms
https://zerotrust-gateway.example.com/v1/signature-agent-card

# Below is an inlined card with the data URL scheme
data:application/json,{"client_name":"Inline Bot","jwks_uri":"https://inline.example.com/.well-known/http-message-signatures-directory"}
~~~

## Formal Syntax

Below is an Augmented Backus-Naur Form (ABNF) description, as described in {{ABNF}}.

The below definition imports `https-URI` from {{HTTP}}, and `dataurl` from
{{DATAURL}}.

~~~
registry = *(cardendpointline / emptyline)
cardendpointline = (
    https-URI /      ; As defined in Section 4.2.2 of RFC 9110
    dataurl          ; As defined in Section 3 of RFC 2397
) EOL


comment = "#" *(UTF8-char-noctl / WS / "#")
emptyline = EOL
EOL = *WS [comment] NL ; end-of-line may have
                       ; optional trailing comment
NL = %x0D / %x0A / %x0D.0A
WS = %x20 / %x09

; UTF8 derived from RFC 3629, but excluding control characters

UTF8-char-noctl = UTF8-1-noctl / UTF8-2 / UTF8-3 / UTF8-4
UTF8-1-noctl = %x21 / %x22 / %x24-7F ; excluding control, space, "#"
UTF8-2 = %xC2-DF UTF8-tail
UTF8-3 = %xE0 %xA0-BF UTF8-tail / %xE1-EC 2UTF8-tail /
         %xED %x80-9F UTF8-tail / %xEE-EF 2UTF8-tail
UTF8-4 = %xF0 %x90-BF 2UTF8-tail / %xF1-F3 3UTF8-tail /
         %xF4 %x80-8F 2UTF8-tail

UTF8-tail = %x80-BF
~~~

## Out-of-band communication between client and origin

A signature agent MAY submit their signature agent card to an origin, or the
origin MAY manually add them to their local registry.

## Registry Endpoint {#registry-endpoint}

A registry MAY be provided via a GitHub repository, a public file server, or a
dedicated endpoint.

The registry SHOULD be served over HTTPS.

A client application SHOULD validate the registry format and reject malformed
entries.

### Authentication {#registry-authentication}

A registry endpoint MAY require authentication to restrict access to authorized
clients. This allows a registry operator to expose registries without revealing
their contents publicly.

No specific authentication mechanism is mandated. Implementations MAY use
pre-shared keys as mentioned in {{PSK-TLS}}, bearer tokens as defined in {{OAUTH-BEARER}}, or HTTP Message
Signatures as defined in {{HTTP-MESSAGE-SIGNATURES}}. HTTP Message Signatures
are RECOMMENDED when key rotation without out-of-band coordination is desired.

### Efficient Polling with Conditional Requests {#registry-conditional-requests}

Registry servers SHOULD include `ETag` and `Last-Modified` response header
fields as defined in {{Section 8.8 of HTTP}}.

Clients SHOULD send `If-None-Match` or `If-Modified-Since` precondition
header fields as defined in {{Section 13.1 of HTTP}} on subsequent requests.
A server SHOULD respond with `304 (Not Modified)` when the registry has not
changed, avoiding redundant data transfer.

### Caching {#registry-caching}

Registry servers SHOULD include a `Cache-Control` response header field as
defined in {{HTTP-CACHE}} to communicate the intended freshness lifetime of the
registry content. Clients SHOULD respect these cache directives and SHOULD NOT
poll more frequently than indicated.

## Signature-Agent header {#signature-agent-header}

Signature Agent Card format defined in {{signature-agent-card}} extends the
format of `Signature-Agent` header as defined in {{Section 4.1 of DIRECTORY}}.

When used for HTTP Message Signatures, a Signature Agent Card MAY be discovered
via a `Signature-Agent` header. The URI carried in the header MAY reference
either an HTTP Message Signatures Directory as defined in {{DIRECTORY}}, or the
`client_id` of a Signature Agent Card resolved as described in
{{cimd-discovery}}.

TODO: a verifier needs a rule to tell the two apart from a single URI. One option
is to disambiguate on the well-known directory path: a URI at the
HTTP Message Signatures Directory well-known location {{DIRECTORY}} is fetched
directly as a directory, and any other URI is treated as a `client_id` and
resolved as a Signature Agent Card per {{cimd-discovery}}. This still requires
parsing the response to confirm which resource was returned, and is under
discussion.

## Composable discovery with Signature-Key {#signature-key}

The discovery in this document is anchored on the `client_id` URL and is
independent of the header that carries it. {{SIGNATURE-KEY}} defines a composable
`Signature-Key` header with an extensible registry of key-distribution schemes.
A future scheme could carry a `client_id` URL, letting a verifier resolve a
Signature Agent Card per-signature alongside the keying material. This document
does not define such a scheme; it is noted as a possible composable carrier.

## Change Notification {#change-notification}

Pull-based consumption with conditional requests is sufficient for most
deployments. When lower notification latency is required (e.g., to promptly
act on entry removal), a registry operator MAY implement a push-based change
notification mechanism.

### Advertising the Notification Endpoint {#notification-advertisement}

A registry operator that supports change notifications SHOULD advertise its
notification endpoint in the registry HTTP response using a `Link` header field
as defined in {{WEB-LINKING}}:

~~~
Link: <https://registry.example/v1/registry-changes>; rel="registry-changes"
~~~

The `registry-changes` link relation identifies an endpoint to which clients
may register callbacks out of band.

TODO: Register the `registry-changes` link relation with IANA, or replace it
with an extension relation URI.

### Callback Registration {#notification-registration}

Registration of a client callback URL with the registry operator is performed
out of band. No specific registration protocol is defined by this document.
An unsubscribe mechanism SHOULD be considered, and MAY also be out of band.

### Notification Requests {#notification-requests}

When an entry is added to or removed from the registry, the registry operator
MUST send an HTTP request to each registered callback URL.

The format in {{CDDL}} is as follows:

~~~cddl
Action = "put" / "delete"

Notification = {
  action: Action,
  signature-agent: tstr
}
~~~

TODO: should we use application/json, a new media-type, permit binary encoding?
TODO: should signature-agent be an array?

`action` denotes the operation performed on the registry:
1. `put` when a new entry has been added,
2. `delete` when an entry has been removed.

The request MUST use `POST`.
It MUST be signed using {{HTTP-MESSAGE-SIGNATURES}} with a key present in the
registry operator's existing signature agent card. This is the signature agent
card associated with the registry operator during out-of-band callback
registration.

The `Content-Type` SHOULD be `application/json`.

Example notification for an added entry:

~~~
POST /registry-callback HTTP/1.1
Host: origin.example
Content-Type: application/json
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest"); \
  created=1741046400; keyid="NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw"
Signature: sig1=:base64signature:
Content-Digest: sha-256=:base64hash:

{
  "action": "put",
  "signature-agent": "https://abc123.registry.example/.well-known/signature-agent-card"
}
~~~

### Notification Processing {#notification-processing}

Upon receiving a notification, the client MUST verify the HTTP Message
Signature against the registry operator's key, discovered via the registry
operator's signature-agent card. Notifications with missing or invalid
signatures MUST be rejected.

A notification is advisory only. The registry endpoint remains the authoritative
source of truth. After verifying a `put` notification, clients SHOULD re-fetch
the affected signature agent card to obtain current metadata.

For `delete` notifications, clients SHOULD confirm the removal by re-fetching
the full registry. This confirmation does not need to happen synchronously with
notification processing.

Registry operators SHOULD retry delivery of failed notifications with
exponential backoff. Clients that miss notifications will recover on their next
conditional pull from the registry endpoint.

# Security Considerations

Malicious actors may put properties which are not theirs in the registry. If signatures are present, clients MUST verify them.
Clients SHOULD reject cards with invalid signatures.

## Registry Integrity

When a registry is served over HTTPS, TLS provides channel integrity between
the server and the client. To additionally bind the registry contents to the
registry operator's cryptographic identity, registry servers SHOULD sign their
HTTP responses using {{HTTP-MESSAGE-SIGNATURES}} with a key present in the
registry operator's own signature agent card. Clients SHOULD verify such
signatures when present.

## Binding Delegated Keys to the Directory Authority

When a signature agent card delegates key discovery using `jwks_uri`, clients
SHOULD validate the referenced directory response to ensure the authenticity
and integrity of the delegated key material.

When a directory server provides key material over HTTP or HTTPS, it is
RECOMMENDED that it include one HTTP Message Signature per key in the response,
with each key used to provide one signature. The signature SHOULD cover
`@authority` with the `req` flag set, and SHOULD include the `created`,
`expires`, `keyid`, and `tag` signature parameters. The `tag` value MUST be
`http-message-signatures-directory`.

Clients SHOULD validate these signatures using the keys provided by the
directory. Clients SHOULD ignore keys from a directory response that do not have
a corresponding valid signature binding the key material to the directory
authority.

## Private Registries

Registry endpoints that require authentication as described in
{{registry-authentication}} limit exposure of registry entries to authorized
clients only. Registry operators SHOULD use authenticated endpoints when the
enumeration of their registry entries is sensitive.

# Privacy Considerations

TODO

## Access Patterns

Registry servers SHOULD avoid logging personally identifiable information from
client requests. Clients fetching a registry reveal their interest in its
entries; registry servers SHOULD treat access logs as sensitive.


# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registration

IANA is requested to register the following parameter in the "OAuth Dynamic
Client Registration Metadata" registry established by {{DCR}}.

**Client Metadata Name:**
: web_bot_auth

**Client Metadata Description:**
: A JSON object carrying bot-specific metadata for a Signature Agent. Its members
are registered in the "Web Bot Auth Metadata Parameters" registry.

**Change Controller:**
: IETF

**Reference:**
: {{web-bot-auth-extension}}

The remaining parameters used by a Signature Agent Card (`client_id`,
`client_name`, `client_uri`, `logo_uri`, `contacts`, `jwks_uri`, `jwks`) are
already registered OAuth client metadata parameters and are not registered by
this document.

## Web Bot Auth Metadata Parameters Registry

IANA is requested to create the "Web Bot Auth Metadata Parameters" registry. This
registry governs the members of the `web_bot_auth` object defined in
{{web-bot-auth-extension}}.

### Registration template

New registrations need to list the following attributes:

**Parameter Name:**
: The name requested (e.g. "purpose"). This name is
case sensitive.  Names may not match other registered names in a
case-insensitive manner unless the Designated Experts state that
there is a compelling reason to allow an exception

**Parameter Description:**
: Brief description of the parameter

**Change Controller:**
: For Standards Track RFCs, list the "IESG".  For others, give the
name of the responsible party.  Other details (e.g., postal
address, email address, home page URI) may also be included.

**Reference:**
: Where this parameter is defined

**Notes:**
: Any notes associated with the entry
{: spacing="compact"}


New entries in this registry are subject to the Specification Required
registration policy ({{!RFC8126, Section 4.6}}). Designated experts can reject
registrations on the basis that they do not meet the security and privacy
requirements defined in TODO.

### Initial Registry content

This section registers the `web_bot_auth` member names defined
in {{web-bot-auth-extension}} in this registry.

#### Expected User Agent Parameter

**Parameter Name:**
: expected-user-agent

**Parameter Description:**
: String or fragment patterns

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-user-agent}}

**Notes:**
: N/A

#### RFC9309 Product Token Parameter

**Parameter Name:**
: rfc9309-product-token

**Parameter Description:**
: Robots.txt product token your signature-agent satisfies.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-robotstxt-token}}

**Notes:**
: N/A

#### RFC9309 Compliance Parameter

**Parameter Name:**
: rfc9309-compliance

**Parameter Description:**
: Does your signature-agent respect robots.txt.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-robotstxt-compliance}}

**Notes:**
: N/A

#### Trigger Parameter

**Parameter Name:**
: trigger

**Parameter Description:**
: Fetcher/Crawler

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-trigger}}

**Notes:**
: N/A

#### Purpose Parameter

**Parameter Name:**
: purpose

**Parameter Description:**
: Intended use for the collected data

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-purpose}}

**Notes:**
: N/A

#### Targeted Content Parameter

**Parameter Name:**
: targeted-content

**Parameter Description:**
: Type of data your agent seeks

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-targeted-content}}

**Notes:**
: N/A

#### Rate control Parameter

**Parameter Name:**
: rate-control

**Parameter Description:**
: How can an origin control your crawl rate

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-rate-control}}

**Notes:**
: N/A

#### Rate expectation Parameter

**Parameter Name:**
: rate-expectation

**Parameter Description:**
: Expected traffic and intensity

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-rate-expectation}}

**Notes:**
: N/A

#### Known URLs Parameter

**Parameter Name:**
: known-urls

**Parameter Description:**
: Predictable endpoint accessed

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-known-urls}}

**Notes:**
: N/A

--- back

# Test Vectors

TODO

# Implementations

TODO


# Acknowledgments
{:numbered="false"}

TODO

The editor would also like to thank the following individuals (listed in alphabetical order) for feedback, insight, and implementation of this document -


# Changelog
{:numbered="false"}

v03

- Reframe Signature Agent Card as an OAuth client metadata document ({{DCR}}/{{CIMD}} namespace) extended with a single `web_bot_auth` object
- Add resolvable `client_id` parameter and CIMD-based discovery (GET, 200 OK, no redirects, client_id match)
- Allow `Signature-Agent` header and registry entries to reference a `client_id` URL
- Register `web_bot_auth` in the OAuth Dynamic Client Registration Metadata registry; move bot-specific parameters into a new "Web Bot Auth Metadata Parameters" registry
- Stop redefining shared client metadata fields (client_name, client_uri, logo_uri, contacts); reference RFC 7591 instead, and drop the non-standard data:text/plain restriction on `client_uri`
- Rename the inline key parameter from `keys` to `jwks` to match the {{DCR}} namespace, and state that the `jwks_uri` resource is a JWK Set (no mandated media type) so generic consumers can read directory keys
- Require `client_id` in a card resolved through its `client_id`
- Restrict registry entries and resolvable cards to `https` (drop `http`)
- Mirror the directory signing requirement: the `jwks_uri` resource SHOULD be signed using HTTP Message Signatures
- Note Signature-Key as a possible composable carrier (informative)

v02

- Add `ips_uri` parameter so client can expose IP addresses
- Add optional `jwks_uri` parameter to separate key material from metadata
- Fix inline data URL example in registry to use valid signature agent card
- Rename "Public list" to "Registry Endpoint" for clarity
- Add authentication guidance for private registry endpoints
- Add conditional GET (ETag/If-Modified-Since) guidance for efficient polling
- Add caching guidance referencing HTTP-CACHE
- Add change notification section: PUT/DELETE webhook signed with HTTP Message Signatures, OOB callback registration, pull as source of truth
- Expand Security Considerations: registry integrity via HTTP Message Signatures, private registry guidance
- Expand Privacy Considerations: customer list exposure, access patterns
- Add normative reference to RFC 8288 (Web Linking)

v01

- Add contributors
- Aligning registry draft with oauth dynamic client registration iana registry
- Add an about-url field
- Add ABNF for discovery of signature-agent card (registry)
- Add precisions about known URLs
- Add placeholder for image size

v00

- Initial draft
