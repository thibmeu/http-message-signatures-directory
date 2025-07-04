---
title: "Signature-Agent and Max-Crawl-rate for Robot Exclusion Protocol"
abbrev: "Signature-Agent and Max-Crawl-rate for Robot Exclusion Protocol"
category: std

docname: draft-meunier-signature-agent-rep-latest
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
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-signature-agent-rep.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  BOT-AUTH: I-D.draft-meunier-web-bot-auth-architecture
  RFC9110:
  RFC9309:
  SIGNATURE-DIRECTORY: I-D.draft-meunier-http-message-signatures-directory

informative:


--- abstract

This document describes a new directive to allow Signature-Agent ({{Section 4 of SIGNATURE-DIRECTORY}}) directive in the Robot Exclusion Protocol ({{RFC9309}}).


--- middle

# Introduction

Bots are increasigly using Signature-Agent as a way to convey identity.
As such, there is interest from Origins to define robot policy based on this header.
In addition, it'd be ideal if some sample rate limit could be communicated.

This documents extends Robot Exclusion Protocol to support these, by extending the user-agent group with new rules.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Specification

## Protocol Definition

Group
: One or more user-agent lines that are followed by one or more signature-agents, max-crawl-rate, and rules. The group is terminated by a user-agent line or end of file. The last group may have no rules, which means it implicitly allows everything.

## Formal Syntax

Based on the formal syntax defined in {{Section 2.2 of RFC9309}}

~~~
 group = startgroupline *(startgroupline / emptyline) ; We start with a user-agent line and possibly more
         *(signatureagentline / emptyline)            ; Specification for signature-agent
         *(maxcrawlrateline / emptyline)              ;
         *(rule / emptyline)                          ; followed by rules relevant for the preceding lines

modifierline = signature-agent-line ; a modifier can either be multiple things, or both.

signatureagentline = *WS "signature-agent" *WS ":" *WS directory-token EOL
maxcrawlrateline = *WS 1*DIGIT *WS ["/" *WS timeunit] *WS

directory-token = DQUOTE "https://" fqdn DQUOTE
timeunit = "s"/"m"/"h"/"d"/"w"

fqdn = ... ; domain as defined by signature-agent. TBD

DQUOTE = "\""
~~~

### Signature-Agent line

Crawlers set their own identity, which is called a directory-agent, to find relevant groups.
The directory token MUST contain only lowercase letters ("a-z"), underscores ("_"), hyphens ("-"), and dots (".").
The directory token SHOULD be a valid FQDN suffix of the identification string that the crawler sends to the service.
For example, in the case of HTTP {{RFC9110}}, the product token SHOULD be a substring in the Signature-Agent header.
The identification string SHOULD describe the public cryptographic key material of the crawler.
Here's an example of a Signature-Agent HTTP request header with a link pointing to a page describing the purpose of the crawler.example.com crawler, which appears as a suffix in the Signature-Agent HTTP header and as a directory token in the robots.txt directory-agent line

~~~
+==========================================+==============================+
| Signature-Agent HTTP header              | robots.txt signature-agent   |
|                                          | line                         |
+==========================================+==============================+
| Signature-Agent: crawler.example.com     | signature-agent: example.com |
+------------------------------------------+------------------------------+
~~~

### Max-Crawl-Rate line

The max-crawl-rate directive specifies the maximum number of requests that a robot SHOULD make to the origin server, for the group it applies to.
Well-behaved agents are expected to comply by limiting their request rate accordingly.
This directive does not enforce technical access restrictions, and adherence is voluntary.
Servers MAY monitor agents behavior and take measures if necessary to protect resources.

### Sitemaps

Should sitemaps be added here?

# Security Considerations {#security}

Signature-Agent group shares the security consideration of {{RFC9309}}. In addition, given Signature-Agent MAY present
a domain name identifying crawlers public cryptographic key material, implementors should treat the content of signature-agent
line as possibly sensitive.


# IANA Considerations

This document has no IANA actions.


--- back

# Examples

## Signature-Agent on example.com

~~~
Signature-Agent: example.com
Allow: *
~~~

## Signature-Agent with raw keyid

Not in the draft yet. We don't want to incentivise not rotating public keys.

~~~
Signature-Agent: poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U
Disallow: /path/to/resource
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
