---
title: "Using KYAPay Tokens"
#abbrev: ""
category: std

docname: draft-skyfire-oauth-using-kyapay-tokens-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
#number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - agent
 - agentic commerce
 - agentic
 - commerce
 - identity
 - JWT
 - bot management
 - fraud
 - account takeover
 - payment
venue:
  github: "skyfire-xyz/draft-skyfire-oauth-using-kyapay-tokens"
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  latest: "https://skyfire-xyz.github.io/draft-skyfire-oauth-using-kyapay-tokens/draft-skyfire-oauth-using-kyapay-tokens.html"

author:
-
  name: Ankit Agarwal
  organization: Skyfire Systems Inc.
  email: ankit_agarwal@yahoo.com
  uri: https://skyfire.xyz
-
  name: Michael B. Jones
  organization: Self-Issued Consulting
  email: michael_b_jones@hotmail.com
  uri: https://self-issued.info/
-
  name: Srinivasa Thumma
  organization: Akamai
  email: sthumma@akamai.com

# see https://github.com/cabo/kramdown-rfc/wiki/Syntax2#authors-contributors

normative:
  RFC2119:
  RFC8174:
  RFC7515: # JWS
  RFC7519: # JWT
  RFC8725: # JWT BCP (BCP 225)
  RFC8693: # OAuth 2.0 Token Exchange
  RFC9110: # HTTP Semantics
  RFC9651: # Structured Field Values
  RFC9421: # HTTP Message Signatures
  RFC7800: # JWT proof-of-possession (cnf)
  RFC9530: # Digest Fields
  RFC6585: # 431 Request Header Fields Too Large
  I-D.skyfire-oauth-kyapay-token:
  I-D.skyfire-oauth-kyapay-token-exchange:

informative:
  RFC6749: # OAuth 2.0
  RFC8176: # AMR values
  RFC8705: # OAuth 2.0 Mutual-TLS
  RFC8792: # Line wrapping in examples
  RFC9457: # Problem Details
  RFC9470: # OAuth 2.0 Step Up Authentication Challenge
  RFC9449: # DPoP
  RFC9111: # HTTP Caching
  RFC9901: # SD-JWT
  I-D.skyfire-oauth-amr-values:
  I-D.skyfire-oauth-id-verification:
  I-D.skyfire-oauth-aml-methods:
  I-D.ietf-oauth-client-id-metadata-document:
  NIST-800-63A:
    title: "Digital Identity Guidelines: Identity Proofing and Enrollment (NIST SP 800-63A)"
    target: https://pages.nist.gov/800-63-3/sp800-63a.html
    date: false
    author:
      - org: National Institute of Standards and Technology
  MCP:
    target: https://modelcontextprotocol.io/specification/2025-11-25
    title: Model Context Protocol Specification
    date: November 25, 2025
    author:
      - org: "Anthropic and the Model Context Protocol Contributors"
  A2A:
    title: Agent2Agent (A2A) Protocol Specification
    date: 2025
    target: https://a2a-protocol.org/latest/specification
    author:
      - org: A2A Project
  UCP:
    title: Universal Commerce Protocol (UCP) Specification
    date: 2025
    target: https://ucp.dev/latest/specification/overview/
    author:
      - org: UCP Contributors
  VINTENT:
    title: Verifiable Intent Specification
    date: 2025
    target: https://verifiableintent.dev/spec/
    author:
      - org: Verifiable Intent Contributors
  SPIFFE:
    title: "Secure Production Identity Framework for Everyone (SPIFFE)"
    target: "https://spiffe.io/docs/latest/spiffe-about/overview/"
    date: false
    author:
      - org: "SPIFFE Project / Cloud Native Computing Foundation"
  KYAPAY-ORG:
    title: "KYAPay: Verified Agent Identity and Payments"
    target: "https://kyapay.org/"
    date: false
    author:
      - org: "Skyfire Systems Inc."
  OpenID.Core:
    target: https://openid.net/specs/openid-connect-core-1_0.html
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    date: 2023-12-15
    author:
      - name: Nat Sakimura
      - name: John Bradley
      - name: Michael B. Jones
      - name: Breno de Medeiros
      - name: Chuck Mortimore

...

--- abstract

The KYAPay Token is a JSON Web Token (JWT) that carries verified identity ("Know Your Agent", KYA) and payment (PAY) information for requests made by software agents on behalf of human principals.
This document describes how security intermediaries -- bot managers, fraud managers, account-takeover (ATO) protection systems, and customer identity and access management (CIAM) systems -- consume KYAPay tokens to answer a question that traditional bot detection cannot: "did a verified human authorize this agent?", rather than "is this a human?".
It specifies how KYAPay tokens are carried in HTTP requests, how they are validated (including in combination with request-signing layers such as HTTP Message Signatures), and how the verified, layered identity in a token is used to make access, routing, fraud detection, account-lifecycle, and step-up decisions.
It defines the token-consuming "recipient" role that the KYAPay Token leaves unspecified.
It is intentionally non-prescriptive about how tokens are created, because agent architectures, agent-identity technologies, and agent-communication protocols are diverse and still emerging;
the token itself is the interoperability contract.

--- middle

# Introduction

Legitimate automated traffic acting on behalf of users is a long-standing feature of the web. Financial aggregation tools, automated booking agents, and data synchronizers routinely interact with web services at the direction of their subscribers. To prevent legitimate automation from being blocked by defenses designed to detect malicious bots, security intermediaries have traditionally relied on a combination of static network metadata -- including published IP ranges, Autonomous System Numbers (ASNs), HTTP header name/value pairs, use of a specific header name, or matching keyword in the User-Agent strings -- to identify bots that serve a legitimate and valuable purpose and apply configured policies.

This model identifies bots primarily by associating their traffic with a known operator, network, and/or an HTTP header signature.  However, its verification mechanisms are brittle: HTTP headers are easily forged, IP ranges change across cloud providers and shared egress infrastructure, and network metadata cannot establish which individual subscriber authorized a particular request. Worst yet, not all legitimate bots identify themselves through these methods, resulting in traffic being blocked by security intermediaries running a bot detection layer. Traditional bot identification can only approximately establish where automated traffic comes from, but not which principal it is acting for.

The rise of capable AI agents makes this limitation more significant. An account aggregation service reimplemented as an LLM-orchestrated agent may perform the same task for the same subscriber, but through dynamic execution flows. The relevant distinction is therefore not simply between human and automated traffic, but between:

* direct human interaction,
* a human or organization acting through an authorized agent, and
* unattended automation without verifiable authorization from an identified principal.

Rather than relying on fragile network indicators to establish legitimacy, an agent presenting a KYAPay token can provide cryptographically verifiable assertions about the agent and the principal it is authorized to represent.

KYAPay tokens provide issuer-signed assertions that can be verified against trusted public keys. A KYAPay token is a signed JSON Web Token (JWT) {{RFC7519}} {{RFC7515}} that can carry the agent instance (`aid`), the execution platform (`apd`), and the identified principal the issuer asserts the agent acts for (`hid`) -- the KYA ("Know Your Agent") information -- and optionally payment credentials (the PAY information). A validated KYA token asserts that a trusted issuer has verified, to an established level of assurance, that the identified principal authorized the identified agent, running on the identified platform, to act on their behalf. A validated PAY token further asserts that a trusted issuer has authorized the agent to execute a payment within specified parameters.

Crucially, identification is distinct from admission. Recipients consume KYAPay tokens as authenticated context for site-configured policy engines, not as an automatic grant of access. A design goal of this specification is that recipients can reliably distinguish attributable, human-authorized agentic traffic from unattributed automation, so that site operators can apply appropriate policy -- admit, rate-limit, require step-up, or deny -- based on verified identity rather than default blocking.

## Scope

This document describes how *consumers* of KYAPay tokens -- in particular the security intermediaries that guard websites, APIs, and accounts -- convey, validate, and act on KYAPay tokens.
It does not redefine the token;
the claims, their meanings, and the base validation rules are specified in the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}} and are referenced normatively here.

This document is deliberately **normative about consumption** and **non-prescriptive about creation**.
{{SecInt}} places requirements on how intermediaries validate and use tokens.
{{TokCreat}} discusses token creation as a software-architecture problem and intentionally does not mandate a mechanism;
the rationale is given in {{NonPrescCreat}}.

## Relationship to Other Documents

This document builds directly on the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}}, which defines the KYA, PAY, and KYA-PAY token types, the claim schema (including the layered identity claims `hid`, `apd`, and `aid`), and the core token validation procedure.
The specification defines the token and the roles of issuer, initiator, and target, but it does not define the role of the party that receives and validates a token in order to make a security decision.
This document defines that role -- the *recipient* (or relying party) -- and specifies its behavior.

Three related specifications define JWT claims and values that a recipient can use to reason about how a principal was authenticated and verified, and that MAY appear in KYAPay tokens:
the Additional Authentication Method Reference Values specification {{I-D.skyfire-oauth-amr-values}}, which defines additional `amr` {{RFC8176}} claim values,
the Identity Verification Methods Values specification {{I-D.skyfire-oauth-id-verification}}, which defines the `ivm` claim and values, and
the Anti-Money Laundering Methods Values specification {{I-D.skyfire-oauth-aml-methods}}, which defines the `aml` claim and values.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the Initiator/Target and agent identity terms Initiator Agent, Initiator Principal, Agent Platform,
and the identity grouping claims `hid`, `apd`, and `aid` defined by {{I-D.skyfire-oauth-kyapay-token}}.

This document defines the following terms:

{:vspace}
**Human-via-agent request**:
: An HTTP request or protocol message issued by an agent that carries a valid KYAPay token attesting that the agent is acting on behalf of an identified, authorizing human principal (an individual or an organization).
  The principal need not be present at the time of the request: the agent may be acting on an instruction given earlier, or on a standing authorization that the agent itself triggers, for example on a price change or on a schedule.

{:vspace}
**Bot**:
: Unattended automation that is not acting on behalf of, or with the authorization of, an identified human principal, and that does not present a valid KYAPay token.
  This document does not attempt to further classify bots;
  existing "good bot" / "bad bot" mechanisms are orthogonal and continue to apply.

{:vspace}
**Recipient (Relying Party)**:
: The entity that receives a KYAPay token, validates it, and acts on the result.
  A recipient may be a security intermediary (below), a Target's own resource server, or an edge/CDN provider acting on a Target's behalf.

{:vspace}
**Security intermediary**:
: A recipient, typically operated by a security vendor or by the Target, that inspects requests and makes an admission, routing, scoring, or lifecycle decision before or as the request reaches the Target's application.
  This document addresses four common (and often overlapping) kinds of security intermediary:

  {:vspace}
  **Bot manager**:
  : An inline application-layer system that detects automated traffic -- using behavioral telemetry, fingerprinting, and network-derived signals -- to decide whether to admit, challenge, throttle, or block a request.

  {:vspace}
  **Fraud manager**:
  : An application-layer system that scores the risk of a transaction or action, typically using identity, reputation, behavioral, and device signals.

  {:vspace}
  **Account-takeover (ATO) protector**:
  : A system that detects and prevents unauthorized access to existing accounts, for example by distinguishing legitimate sessions from hijacked or automated ones.

  {:vspace}
  **CIAM (customer identity and access management) system**:
  : A system that manages account creation, authentication, and login for a Target's customers.

{:vspace}
**Human presence signal**:
: The aggregate assurance, derived from a validated KYAPay token, that a request is authorized by an identified human principal.
  See {{HumanPresence}}.

{:vspace}
**Assurance level**:
: A per-entity indication of how strongly an identity in the token was verified (for example, the identity-proofing level of the human principal).
  See {{HumanPresence}}.

# Design Principles {#DesignPrinc}

This section is informative.
It records the assumptions that motivate the normative requirements in later sections.

## Human Identity is Paramount

An underlying hypothesis of KYAPay is that commerce -- with or without payment -- is conducted between humans and human organizations, not between machines.
An agent is a tool through which a human or organization acts.
Consequently, the identity that matters most is the human identity behind the request, and liability for a request must ultimately be assignable to a human party: the principal, the developer of the agent, the operator of the agent platform, or some other identified party.
KYAPay tokens are structured to convey these identities so that this auditable, cryptographically verifiable chain of responsibility -- from agent action back to an accountable human -- can be established.

## Ubiquity of Access on the Existing Web

Agents are useful in proportion to the breadth of the web they can act across.
If a human must pre-configure access for an agent at each site before the agent can act, the result is indistinguishable from the workflow automation that already exists, and the principal loses the probabilistic discovery -- of new merchants, products, services, and alternatives -- that is one of the main reasons to use an agent at all.
The goal is therefore agentic access to the existing web, not a parallel, agent-only surface that must be set up in advance.
This goal can only be met if security intermediaries admit legitimate human-via-agent requests without per-site setup when a valid token is present and the verified identity it carries meets the site's policy for the requested action ({{BotMgr}}).

## Works with Today's Web at Zero Merchant Tech Lift

Websites exist today, in the millions, and merchants and services already know how to sell through them, manage fraud, make offers, build brand loyalty, and test changes.
A key property of a website (and, similarly, of a merchant-controlled MCP {{MCP}} server) is that the merchant controls its own interface and is not locked into an API schema that must be carefully versioned;
agents driven by large language models (LLMs) can frequently learn to use these interfaces automatically.
A protocol for delivering agent identity and payment credentials should therefore work with today's web without requiring merchants to build new APIs.
KYAPay meets this by conveying credentials in the HTTP request itself ({{ConveyingRequests}}), which requires no change to how a merchant's site is built.
This holds where the party attaching the token controls the header fields of the request that reaches the Target.
Where it does not, the token is conveyed by another binding ({{OtherInterfaces}}) or the interaction does not use this header field.

## One Source of Truth Across All Layers of Defense

Bot managers, fraud managers, ATO protectors, and CIAM systems all operate at the application layer.
Today, each makes its decision in isolation, and their verdicts on the same request can contradict one another.
A single validated KYAPay token gives every layer the same verified identity and, where present, payment context, so that their decisions are more likely to be consistent than conflicting.

## Runtime Business Decisions

A further goal is to give the Target the information it needs to make a business decision at request time: do I want to do business with this human, arriving via this agent, on this platform, for this action?
Because the token carries verified, layered identity and (optionally) payment context, the Target and its intermediaries can decide at runtime whether to admit, price, personalize, step up, or decline.
As one illustration, a merchant that recognizes a KYAPay-bearing request MAY adapt its existing pages for agent consumption using the same techniques it already uses to restyle pages, retaining ownership of its interface and its existing fraud and good/bad-actor signals rather than exposing a separate API surface.

# The Trust Stack: Bearer Tokens Today, Proof of Possession Ahead {#TrustStack}

This section is informative.
It situates KYAPay tokens among the other mechanisms a recipient encounters and, importantly, distinguishes what is deployed today from the planned evolution, so that the consumption requirements in {{SecInt}} are read against the correct baseline.

## KYA is an Issuer-Signed Bearer Token Today {#Bearer}

As currently deployed, a KYA (or KYA-PAY) token is an issuer-signed, short-lived bearer token.
Its single most valuable operational property is that a Target has to do almost nothing to consume it: verify a JWT against the issuer's public key via JWKS, exactly as it would verify any OpenID Connect {{OpenID.Core}} or OAuth {{RFC6749}} token.
Any Target with a standard JWT library can participate, including those behind CDNs, API gateways, and serverless platforms.
Preserving that low bar is a primary design goal ({{DesignPrinc}});
mechanisms that would raise it are weighed against the adoption they would cost.

A bearer token establishes identity but not proof of possession: possession of the token is all that is needed to present it, so a token captured within its validity window can be replayed.
Today this residual, in-window risk is bounded rather than eliminated, by means that do not raise the consumption bar:

* short token lifetimes (`exp`), so a captured token expires quickly;
* audience binding ({{Counterparty}}), so a token minted for one Target cannot be presented to another;
* TLS on every hop, which mitigates on-the-wire capture and man-in-the-middle; and
* issuer-gated onboarding (KYC/KYB), which mitigates malicious agents and Targets being admitted to the network in the first place.

The accepted residual threats, and the irreducible ones, are stated in {{SecCon}}.

A KYAPay token is reusable within its validity period and audience.
Unless a recipient knows that the issuer issues a distinct token for each request, it MUST NOT treat repeated presentation of the same token, within that period and audience, as evidence of replay.
Nothing here prevents an issuer from issuing a distinct token for each request where that is practical; this document describes the general case, in which a token is presented more than once.

## Identity and Proof of Possession are Different Layers

It is useful to separate two questions a recipient may want answered:

* *Who is behind this request?*
  This is answered by the KYA token: a trusted, neutral issuer that performs KYC/KYB attests to the identity of the human principal (`hid`), agent platform (`apd`), and agent (`aid`), acting as a client-side trust anchor analogous to a Certificate Authority.
  KYA is a set of verifiable *nouns* -- identity, verification status, authorization, payment -- and, being a JWT, it is transport-agnostic: the same token rides HTTP, WebSockets, gRPC metadata, a message queue, or a stdio MCP channel, and a recipient that needs it for audit or dispute resolution can retain it and re-verify it later ({{Section 5.2 of I-D.skyfire-oauth-kyapay-token}}).

* *Did the sender actually hold the credential's key?*
  This is proof of possession, which a bearer token does not provide.
  Establishing it requires the agent to sign the request itself;
  over HTTP, HTTP Message Signatures {{RFC9421}} are a mechanism for this.
  A signature proves control of a key but says nothing about whose key it is -- it is a *verb*, not an identity format.

Because these are different layers, HTTP Message Signatures {{RFC9421}} and similar request-signing mechanisms are complementary to KYA, not alternatives to it;
request signing supplies authenticity and possession, while KYA supplies identity, authorization, and payment.
A complete proof-of-possession design uses both, bound together, as described next.

A KYA token is verified the same way however a request is routed, because it is a self-contained JWT and not a signature over the request.
A signature over the request is different: it can be verified only where the request still has the form the agent signed.
Requests are routinely rewritten on the way to an origin.
CDNs, load balancers, API gateways and edge workers add or strip path prefixes, route to internal host names and normalize URIs, and sites deliberately keep their external and internal URLs separate so as not to expose their internal architecture.
A design in which every recipient verified a signature over the request would fail at each such hop, or need special handling there.
This is why KYA is a bearer token by default ({{Bearer}}), and why, where proof of possession is used, the request signature is verified at the first hop ({{PoPSignatures}}).

## The Proof-of-Possession Path (Planned Evolution) {#PoPPath}

When per-request signing becomes practical at scale, the planned evolution of KYA is a Certificate-Authority-like model that adds proof of possession without changing the trust anchor:

1. the issuer onboards the agent as a CA would;
1. the agent generates a key pair and provides its public key to the issuer;
1. the issuer binds that public key into the KYA token using the JWT confirmation claim `cnf` {{RFC7800}}. The token MUST carry `cnf.jwk` -- the key itself. A key identifier would satisfy {{RFC7800}} only on its own terms, which hold "provided the recipient is able to obtain the identified key": a recipient meeting the agent for the first time holds no such key, and a thumbprint is a one-way hash from which none can be recovered. The issuer cannot determine when that proviso is met -- it knows neither which recipients a token will reach, nor the initiator's intent, nor the state of any interaction between initiator and target. Nor can it rely on continuity between tokens: a recipient may meet any given token first, so each token must stand on its own. Carrying the key is therefore the only form that works in the general case. Other confirmation members MAY additionally be present; a recipient MUST NOT be required to obtain the key by any means other than reading `cnf.jwk`; and
1. the agent signs each request in place using {{RFC9421}}, covering the components listed in {{PoPSignatures}}, and the signature is verified against the key in `cnf`, with no separate proof-of-possession token.

This yields an unbroken chain from request to key to identity: the request is signed by a key, and that key is vouched for as the initiator's by the issuer.
Preferring a request signed against the `cnf` key over a detached, DPoP-style proof {{RFC9449}} keeps a single token -- the recipient validates the KYA token once (per session), caches the `cnf` key, then performs only a fast per-request signature check, instead of validating two artifacts and confirming they are bound to each other.
When present, this binding is consumed as specified in {{ProcModel}} and closes the malicious-Target-reuse risk of the bearer model and stops a stolen token being used on its own ({{SecCon}}).

Adoption remains the constraint on when this becomes mandatory.
Per-request signing is costly: hardware-protected keys are secure but slow, software keys are fast but exfiltratable, and post-quantum algorithms do not make signing faster.
Binding the signature to the request ({{PoPSignatures}}) stops a captured signature from being altered or presented with a different request; an identical request can still be resent until the signature expires, and this document does not rely on per-Target nonce caches to prevent that ({{Replay}}).
Other request-bound designs considered (mutual TLS {{RFC8705}}, TLS channel binding, session-based schemes) each raise the participation bar or add Target-side state.
Until fast, secure signing and a workable anti-replay approach exist that keep the Target's job about as simple as verifying a JWT, KYA remains a bearer token by default and proof of possession is an optional, forward-compatible upgrade rather than a precondition for participation.

## Request Signatures for Proof of Possession {#PoPSignatures}

Where a token carries a proof-of-possession key ({{PoPPath}}), the agent signs each request with that key using HTTP Message Signatures {{RFC9421}}.
The signature MUST cover:

* the `@method` derived component;
* the `@target-uri` derived component;
* the `KYAPay-Token` header field, so that a signature cannot be presented with a different token; and
* the `Content-Digest` header field {{RFC9530}}, whenever the request has content.

The signature MUST carry the `created` and `expires` parameters.

Proof of possession is optional.
A token that carries no proof-of-possession key is a bearer token ({{Bearer}}), and nothing in this section applies to it.

A covered component has to hold the same value when the signature is verified as when it was created, and a hop that rewrites a request changes the components that say where it is going.
The signature is therefore verified by the first recipient to receive the request as the agent sent it, before any rewriting, which is usually the CDN, bot manager, or edge worker in front of the Target.
The first recipient that rewrites a request MUST verify the signature before rewriting it.
The recipient that verifies the signature MUST NOT forward a request whose signature is absent or invalid; it rejects the request ({{ProcModel}}).
Where the signature covers `Content-Digest`, the field MUST be checked against the content {{RFC9530}} before the request is acted on, by the recipient that verifies the signature or by a recipient behind it that receives both the content and the signed `Content-Digest` value unchanged; a mismatch, or a digest that cannot be checked, MUST be treated as an invalid signature.
A recipient behind that hop MAY rely on it instead of verifying the signature itself, but only where the deployment ensures that requests cannot reach it without passing through that hop.
It still validates the token as described in {{ProcModel}}.

The other elements that {{Section 1.4 of RFC9421}} requires a profile to specify, including the source of the verification key, the `keyid` and `tag` parameters, and error responses, are expected to be specified in a future revision of this document.

# Conveying KYAPay Tokens in Requests {#ConveyingRequests}

## The KYAPay-Token HTTP Header Field {#HeaderField}

When KYAPay tokens are used with websites and HTTP APIs, a human-via-agent request carries its token(s) in an HTTP header field, in the same spirit that a human's browser conveys identity and payment details in the request itself.
For a directly interacting human, such details are typically carried in the request body (for example, form fields at checkout);
for a human-via-agent request, they are carried in the `KYAPay-Token` header field defined here.

The `KYAPay-Token` header field is an HTTP request header field whose value is a KYA, PAY, or KYA-PAY token, as defined in {{I-D.skyfire-oauth-kyapay-token}}.
HTTP field names are case-insensitive {{RFC9110}}.

`KYAPay-Token` is a List Structured Field ({{Section 3.1 of RFC9651}}).
Each member is a Token ({{Section 3.3.4 of RFC9651}}) whose value is a JWT, with no parameters.
A compact JWT consists only of base64url characters and periods, and it begins with the encoding of a JSON object, so its first character is a letter; it is therefore a valid Token as it stands.
For example:

    KYAPay-Token: eyJhbGciOiJFUzI1NiIsInR5cCI6Imt5YStqd3QifQ.eyJpc3Mi...

A request MAY convey more than one token (for example, separate KYA and PAY tokens), either as members of one field line or in several `KYAPay-Token` field lines, which are combined before parsing as described in {{Section 4.2 of RFC9651}}.

A sender:

* MUST place exactly one JWT in each list member;
* SHOULD NOT send more than one token of the same `typ` ({{I-D.skyfire-oauth-kyapay-token}}) in a single request; and
* MUST NOT rely on the ordering of tokens within the field to convey meaning.

A recipient:

* MUST determine the type of each token from its `typ` header parameter ({{I-D.skyfire-oauth-kyapay-token}}), not from its position;
* MAY reject a request that carries more than one token of the same `typ`;
* MUST parse the field as a List {{RFC9651}}. A field that fails to parse fails as a whole, and a recipient MUST NOT attempt to recover individual members from it;
* MUST reject the request if the field fails to parse, or if any member is not a Token whose value is a well-formed JWT. {{Section 4.2 of RFC9651}} permits either ignoring a field that fails to parse or treating the message as malformed; this document requires the latter; and
* MUST accept List members of up to the length given in {{TokenSize}}.

Intermediaries and caches MUST treat `KYAPay-Token` as containing sensitive, request-specific credentials;
it MUST NOT be cached in a way that would allow it to be replayed on a different request, and it MUST NOT be logged or forwarded to parties other than legitimate participants (see {{PrivCon}}).

The `KYAPay-Token` header field is intended for use over TLS.
A token MUST NOT be sent over a non-TLS-protected connection.
When the token carries a proof-of-possession key ({{TrustStack}}), it SHOULD be accompanied by a request Message Signature {{RFC9421}} over that key;
otherwise it is a bearer token, subject to the controls and accepted risks in {{SecCon}}.

A CORS preflight request does not carry the `KYAPay-Token` field: the user agent sends it to ask whether the field may be sent.
A recipient MUST NOT apply its token-absent policy to a preflight request.

Where a response depends on the `KYAPay-Token` field, the response MUST include `Vary: KYAPay-Token` and MUST be marked so that shared caches do not store it, for example with `Cache-Control: private`.
The protection in {{Section 3.5 of RFC9111}} applies only to requests that carry the `Authorization` field.

KYAPay tokens are carried in a dedicated header field rather than in `Authorization`.
A Target commonly uses `Authorization` for its own authentication; a KYAPay token is not a credential for access to the Target's resources; and the same token is carried over transports other than HTTP ({{OtherInterfaces}}), where `Authorization` has no meaning.
For the same reasons, a recipient states its requirements in the `Accept-KYAPay-Token` response field ({{AcceptField}}) rather than in a `WWW-Authenticate` challenge.

## Token Size {#TokenSize}

A recipient MUST accept a `KYAPay-Token` field value of at least 6144 octets. The
minimum applies to the field value as a whole, however many tokens it carries, and
a recipient's parser MUST accept List members of that length. A recipient MUST NOT
truncate the field value, and MUST NOT treat a field it rejects for its size as
absent. When it rejects a field for its size, it SHOULD respond with 431 (Request
Header Fields Too Large) {{RFC6585}}. With a typical JOSE header and an ES256
signature, 6144 octets fits a token whose claims total 4096 octets of JSON, the
limit {{I-D.skyfire-oauth-kyapay-token}} asks issuers to observe. It also stays
within common origin-server limits for a single header field.

## Conveying Tokens over Other Interfaces {#OtherInterfaces}

Beyond HTTP requests to websites and APIs, agents interact through a growing set of interfaces and protocols, including agent-to-tool interactions such as MCP {{MCP}}, agent-to-agent interactions such as A2A {{A2A}}, agent-to-API interactions (for example, OpenAPI descriptions converted into locally callable tools), and emerging protocols such as Universal Commerce Protocol {{UCP}} and Verifiable Intent {{VINTENT}}.
KYAPay is designed so that the same self-contained token can be conveyed across any of these interfaces -- as an HTTP header field, a message field, or a tool argument -- to deliver verified identity and payment credentials to the receiving service, API, or agent.
Because a KYA token is a self-contained JWT, it is transport-agnostic in a way that a request-signing mechanism is not: HTTP Message Signatures {{RFC9421}} are meaningful only for HTTP exchanges, whereas the same token can ride HTTP, WebSockets, gRPC metadata, message queues, or a stdio MCP channel, and can be retained by a recipient that needs it for audit or dispute resolution and verified again later ({{Section 5.2 of I-D.skyfire-oauth-kyapay-token}}).
This document specifies only the HTTP header binding in {{HeaderField}};
bindings for other protocols are expected to be defined by those protocols or by companion specifications.
In all cases, the validation and usage rules of {{SecInt}} apply to the token, once received.

## Authenticating the Counterparty {#Counterparty}

The `aud` and `tsi` claims are identifiers assigned by the Token Issuer (the Identifying the Target section of {{I-D.skyfire-oauth-kyapay-token}}); their values are known to a Target that has onboarded to that issuer and are not otherwise discoverable.
The `tdm` claim names the Target's domain, is not issuer-specific, and requires no relationship with the issuer to interpret.
A Target that has onboarded to an issuer checks `aud` and `tsi`, and MAY also check `tdm`; a security intermediary acting for such a Target MAY be configured with that Target's `aud` and `tsi` values and check them.
A recipient that has not onboarded, including a security intermediary acting for many Targets, SHOULD check at least that `tdm` names the domain it is protecting for that request; where `tdm` also names a host or a path, the recipient MAY check those too.
`tdm` MUST be present in tokens whose Target is identified by a domain name.
At the transport and discovery layers, mechanisms such as DNS domain names (with TLS server authentication), A2A agent cards {{A2A}}, and MCP server identification {{MCP}} help an agent confirm that it is communicating with the correct counterparty before presenting a token.

## Attaching Tokens to Requests {#Attaching}

How an agent attaches a token to a request is an implementation matter, for the reasons given in {{TokCreat}}.
Any kind of request may carry a `KYAPay-Token` field; whether it does, and which token it carries, depends on where the request is sent:

* A request to the Target carries the token requested for that Target.

* A Target may rely on other hosts, such as its own hosts in other domains or vendors acting on its behalf.
  A request to such a host carries a token that names the Target in its audience claims ({{Counterparty}}) and carries the claims that apply to that host, which may include payment claims.
  Where no claims are set for the host, the token carries only the base claims: `hid` limited to its `id` member and assurance claims, `apd`, and `aid` without `creation_ip`, with no payment claims.
  Where the claims for the host match those of the Target's token, the agent MAY send the Target's token.

* A request to any other host SHOULD NOT carry a token.
  If it does, the token MUST NOT carry more than the base claims.

This applies however the token is attached. An agent that adds the field below the page, for example by browser-level header injection, applies these rules to each request according to its destination, because a page also sends requests to hosts that are neither the Target nor acting on its behalf.

The Target determines which hosts act on its behalf and which claims each receives.
Until a mechanism exists for the Target to declare this, an agent learns it by configuration or from an arrangement with the Target.

## The Accept-KYAPay-Token Response Field {#AcceptField}

`Accept-KYAPay-Token` is a Dictionary Structured Field ({{Section 3.2 of RFC9651}}) that a recipient sends in a response to state what a request needs to carry for the recipient to accept it.
Each member states one requirement.
The member's key is a label chosen by the recipient.
Its value is an Inner List of Strings, each naming a claim, using the claim names of {{I-D.skyfire-oauth-kyapay-token}} with a period between a claim and its sub-claim (for example, "hid.email").

Parameters on a member qualify the requirement.
Uses of KYAPay vary too widely for this document to define a complete set, so recipients define the parameters and values they need.
Three that are commonly useful are:

* `purpose`: a Token saying why the recipient needs the claims, which an agent can show to the human principal when asking for consent;

* `optional`: a Boolean; when true, the recipient would like the claims but does not require them; and

* `issuers`: a String listing the issuers whose tokens the recipient accepts for this requirement, each as the value it uses in the `iss` claim, separated by single spaces.

For example (with line wrapping per {{RFC8792}}):

    NOTE: '\' line wrapping per RFC 8792

    Accept-KYAPay-Token: checkout=("hid.email");purpose=checkout, \
      fraud=("hid.phone_number");purpose=fraud_prevention;optional

A recipient that requires payment names payment claims in the same way, for example `pay=("amt" "cur");purpose=checkout`.

An agent MUST ignore parameters it does not recognize.
A recipient that responds to cross-origin requests MUST list `Accept-KYAPay-Token` in `Access-Control-Expose-Headers`, so that a browser-based agent can read it.

# Using KYAPay Tokens at Security Intermediaries {#SecInt}

This section is normative.
It defines a common processing model and then describes how each kind of security intermediary uses the validated token.
Throughout, "validate the token" means to perform the validation procedure defined in {{I-D.skyfire-oauth-kyapay-token}}, Section 4 (JWT header validation, signature and issuer validation, and validation of `exp`, `iat`, `jti`, the audience, and `env`), plus the PAY-specific validation for `pay+jwt` and `kya-pay+jwt` tokens.

## Common Processing Model {#ProcModel}

A recipient first decides, from its own policy for the resource and action requested, whether it uses KYAPay tokens for that request.
A recipient that does not MAY ignore the `KYAPay-Token` field.
A recipient that does, on receiving a request that carries one or more KYAPay tokens, processes the tokens of the types it accepts ({{I-D.skyfire-oauth-kyapay-token}}) as follows, and sets aside tokens of other types ({{I-D.skyfire-oauth-kyapay-token}}, Section 4).
A security intermediary:

1. MUST extract each token as described in {{ConveyingRequests}}.
1. MUST confirm that the token's issuer is one the recipient trusts for the claims being relied upon (see {{SecCon}}).
   This check MUST precede any processing that resolves a value taken from the token, and in particular any network retrieval driven by the `iss` claim.
   Until the token has been verified, `iss` is supplied by the sender: resolving it before the issuer is known to be trusted would allow an unauthenticated request to cause an outbound fetch to an arbitrary URL of the sender's choosing.
   The trusted-issuer set is configured out of band and is not derived from the token.
   A recipient MUST NOT retrieve key material for an issuer that is not in its configured set of trusted issuers.
1. MUST validate each token as specified in {{I-D.skyfire-oauth-kyapay-token}}, Section 4.
   A token that fails validation MUST NOT be treated as conveying a human presence signal, and the intermediary MUST decline what it governs for that request, rather than applying its policy for requests that carry no token: a bot manager blocks the request, a fraud manager declines the transaction, or, where it inspects the token only after the action, handles the failure as it handles any other fraud finding on a completed transaction, an ATO protector blocks the sign-in, a CIAM system refuses the account action or token exchange, and a payment service does not proceed with the payment.
   Where the decline is returned to the agent in an HTTP response, that response is as described in {{StepUp}}.
1. MUST verify the token signature against a key obtained from the issuer's JWK Set, discovered via the `iss` claim using the `/.well-known/jwks.json` mechanism ({{I-D.skyfire-oauth-kyapay-token}}).
   Because KYAPay tokens are self-contained, this verification SHOULD be performed locally, without a synchronous callout to the issuer per request;
   issuer keys SHOULD be cached and refreshed according to standard JWK Set practice.
1. MUST confirm that the token is intended for this recipient, as described in {{Counterparty}}.
1. MUST confirm that the `env` claim (for example, `production`) matches the environment the intermediary is operating in.
1. When the token carries a proof-of-possession key (the `cnf` claim {{RFC7800}}, {{PoPPath}}), MUST verify that the request is signed by that key as described in {{PoPSignatures}}, and MUST reject the request, and not forward it, if the signature is absent or invalid.
   An intermediary behind a hop that has already done so MAY rely on that hop instead, as {{PoPSignatures}} describes.
   When the token carries no such key, this step does not apply.
1. When the token carries no such key, it is a bearer token: the intermediary MUST rely on the controls in {{SecCon}} (short lifetime, audience binding, TLS, issuer-gated onboarding) and MUST NOT assume that mere presentation of the token proves the sender holds a key bound to it.
1. SHOULD use the token's claims to inform its decision, as described in the following subsections, only after successful validation.

A security intermediary MUST NOT treat the mere presence of a `KYAPay-Token` header field as a human presence signal;
only a successfully validated token from a trusted issuer conveys that signal.

## Determining Human Presence and Assurance {#HumanPresence}

The core purpose of consuming a KYAPay token is to distinguish a human-via-agent request from a bot, and to gauge how strongly to trust it.
After validation, a recipient SHOULD derive the human presence signal from the layered identity in the token, treating each layer as a distinct entity verified by distinct means:

* the human principal (`hid`): the presence and content of the human identity claim, and how strongly it was verified -- for example a verified status and verifier ({{I-D.skyfire-oauth-kyapay-token}}), and, where a deployment conveys it, an identity-proofing assurance level (such as an Identity Assurance Level per {{NIST-800-63A}}) and the
`amr` {{I-D.skyfire-oauth-amr-values}},
`ivm` {{I-D.skyfire-oauth-id-verification}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims describing how the principal was authenticated and verified;

* the agent platform (`apd`): the identity of the operator that built and runs the agent, and whether it was verified (for example, a business/Know-Your-Business attestation), which supports reputation-based logic about the platform; and

* the agent instance (`aid`): the specific agent, including a meaningful name and, where present, references to its OAuth identity ({{I-D.ietf-oauth-client-id-metadata-document}}) and a proof-of-possession key bound to the token via `cnf` {{RFC7800}} (or a request-signing key published in an A2A Agent Card {{A2A}}).

Because these are different entities verified to different degrees, a recipient SHOULD NOT collapse them into a single yes/no signal.
Instead, it SHOULD apply per-entity, per-action policy.
For example, a recipient might require a higher human-principal assurance level and a "registered" agent for a payment than for browsing:

    /checkout: require hid assurance >= document+biometric
	       require apd verified as a business
	       require aid bound to a verified domain
    /payment:  require hid assurance >= document+biometric
	       require hid authenticated with a hardware key
	       require aid registered by its platform

The strength of the human presence signal is a function of which identities are present, how strongly each was verified, and how much the recipient trusts the attesting issuer and verifier.
A recipient MAY combine the token signal with its existing signals rather than replacing them, and MAY accept a lower assurance for low-risk actions while requiring step-up ({{StepUp}}) for higher-risk ones.

## Bot Managers {#BotMgr}

A bot manager sits inline in the request path to make per-request admission decisions with minimal added latency. Traditionally, it relies on a detection pipeline -- client-side behavioral telemetry, browser fingerprinting, request timing, and network-signal analysis -- to infer whether a request is automated and who operates it.

### Token is present and valid

KYAPay tokens transform this detection model into a self-identification model. By presenting a valid KYA token, an agent explicitly identifies itself as automated up front. The bot manager verifies the token -- checking the cryptographic signature against the issuer's published keys, issuer trust, validity window, and audience -- and classifies the request as a verified agent.

For token-bearing traffic, the identification question is settled: the bot manager does not need to execute its behavioral or fingerprinting detection heuristics to prove the request is automated. Requests that carry no token fall back to the standard detection pipeline; a request whose token fails validation is declined ({{ProcModel}}).

Settling identification, however, does not grant automatic access. Identification is distinct from admission: classifying a request as a verified agent simply provides authenticated context. The bot manager uses the token's verified identity layer -- the human principal (`hid`), the agent platform (`apd`), and the agent instance (`aid`) -- to select and evaluate the site's configured access policy to determine the final disposition (admit, rate-limit, require step-up, or deny) rather than applying bot-blocking challenges designed for unattended, unattributed automation.

Per-entity rate and abuse limits ({{BadActor}}) continue to apply to admitted requests, and a site MAY still challenge, or require step-up ({{StepUp}}) for, sensitive actions.

KYAPay tokens are well-suited to inline bot management for several reasons:

* **Low Latency.**
Because the token is a self-contained, signed JWT, the bot manager performs standard JWT verification locally and needs no per-request callout to a third party. Issuer keys SHOULD be cached.

* **Policy Selection via Identity Context.**
A validated token provides verified initiator identity (and, when targeting an agent, target identity), which the bot manager uses to select and evaluate the appropriate site policy rather than treating the traffic as suspected automation. By contrast, network-origin claims (such as declared source IP ranges) are unreliable as proof of origin -- addresses change and can be masked by VPNs, CGNAT, or shared pools -- so a recipient MAY use them as a weak corroborating signal only, but MUST NOT treat an IP match alone as sufficient to bypass controls nor an IP mismatch as conclusive. Short lifetimes, audience binding, and proof-of-possession ({{TrustStack}}), not IP correlation, are the recommended controls against token misuse.

* **Replay Resistance.**
A bearer token captured within its validity window can be replayed; a bot manager bounds this with short lifetimes, audience binding, and TLS ({{SecCon}}). Where a token carries a proof-of-possession key ({{PoPPath}}), the per-request signature is verified at the first hop, before any rewriting, which is usually the bot manager or the edge/CDN provider acting for the Target ({{PoPSignatures}}); this stops a stolen token being used on its own ({{Replay}}).

### Token present but invalid

In some cases, a KYA token may be present in the request but the validation process may reveal some anomalies.
The bot manager then responds as described in {{StepUp}}, with 403 (Forbidden), and can include the `Accept-KYAPay-Token` field stating what a request needs in order to succeed.

In the examples in this section and the next, the `issuers` parameter ({{AcceptField}}) lists the issuers from which the bot manager accepts tokens, and a problem details body {{RFC9457}} gives the reason.
This document does not define those problem types; they are examples for discussion only.

**Token expired.**
Because tokens have a short validity window, a bot manager may occasionally detect a token that expires during the validation process.
The agent can obtain a new token from one of the listed issuers and retry the request.

    NOTE: '\' line wrapping per RFC 8792

    HTTP/1.1 403 Forbidden
    Content-Type: application/problem+json
    Accept-KYAPay-Token: admit=("hid" "apd" "aid"); \
      issuers="https://issuer.example.com https://iss.example.org"

    {"type": "https://bot-manager.example/problems/expired-token"}

**Token invalid.**
Some anomalies may be detected when validating the token as described in {{ProcModel}} and {{HumanPresence}}.
The response has the same form, with a different problem type:

    {"type": "https://bot-manager.example/problems/invalid-token"}

### Token not present
Organizations and individuals developing AI agents may be unaware of the requirement to identify their traffic using a Know Your Agent (KYA) token. Consequently, bot managers utilizing advanced detection methods may misclassify this agentic traffic as malicious and block it. To prevent a degraded user experience, which can lead to lost business opportunities for the host website, bot managers should issue a custom challenge response instead of an outright block. This proactive approach gently encourages developers to adopt and implement the KYA standard.

When a bot manager detects advanced automation that lacks a Know Your Agent (KYA) token or other standard forms of identification, it responds in the same way, with the problem type `https://bot-manager.example/problems/missing-token`.
After the agent acquires a valid KYA token and reissues the request, the transaction can complete successfully.

This light challenge is designed to help with KYA standard adoption and improve agentic traffic acceptance with merchants and increase agent autonomy in successfully completing tasks on behalf of their user principal.

## Fraud Managers

A fraud manager operates at the application layer and scores the risk of an action or transaction.
It benefits from the token's layered identity and verification metadata:

* It SHOULD incorporate the per-entity verified identity (`hid`, `apd`, `aid`) and assurance levels into its risk model, distinguishing a request backed by a strongly verified human principal on a reputable platform from a weakly verified or anonymous one.

* It SHOULD use the `verifier`/`verified`/`verification_id` sub-claims, and the
`amr` {{I-D.skyfire-oauth-amr-values}}
`ivm` {{I-D.skyfire-oauth-id-verification}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims where present, to weight how the principal was authenticated and verified.

* For `pay+jwt` and `kya-pay+jwt` tokens, it SHOULD use the payment context (`amt`, `cur`, `stp`, `sti`, and the pricing claims `tpr` and `tps`) as additional transaction risk signals, after performing the PAY-token validation of {{I-D.skyfire-oauth-kyapay-token}}, Section 4.
  Because the PAY token pins amount, currency, and target, the fraud manager MAY rely on those as hard limits enforced by the token rather than re-deriving them.

A fraud manager need not sit inline in the request path.
In practice the token is made available to it either inline (the Target forwards the received token to its fraud manager) or out of band (for example, script running on the Target's page captures the `KYAPay-Token` value and sends it to the fraud manager before the request reaches the Target's application).
The same token MAY be inspected before the action, after it, or both.
A recipient that inspects the token after the action evaluates its time-based claims, such as `exp`, against the time it received the request, as recorded by the recipient and not as supplied by the sender.
In all of these arrangements, the token issuer is not in the transaction path;
the fraud manager verifies the self-contained token locally by JWKS, as described in {{ProcModel}}.
The fraud manager MAY continue to apply its existing behavioral and device models;
the token supplements, and does not replace, those signals.

## Responding When a Token Falls Short {#StepUp}

A recipient can decline a request because it carries no KYAPay token, because the token fails validation ({{ProcModel}}), or because the token is valid but does not carry what the action requires: for example identity claims, a higher assurance level for a sensitive action, or a payment token.
Not all actions carry the same risk; viewing a loyalty balance is far less sensitive than making a large payment, and a token attests only the assurance established when it was issued.

Where the decline is returned to the agent in an HTTP response, that response is 403 (Forbidden), or 402 (Payment Required) where what is missing or insufficient is payment, and MAY include the `Accept-KYAPay-Token` field ({{AcceptField}}) stating what a request needs in order to succeed ({{DisclosingRequirements}}).
Where the request is to an endpoint whose protocol specifies its own error responses, the recipient instead responds as that protocol specifies, and MAY include the `Accept-KYAPay-Token` field; for example, an OAuth 2.0 authorization server that declines a token exchange responds as specified in {{Section 2.2.2 of RFC8693}}.
Over other interfaces ({{OtherInterfaces}}), how a decline is signaled is left to the binding for that interface.
A recipient SHOULD NOT use 401 (Unauthorized) for these cases: that status requires a `WWW-Authenticate` challenge ({{Section 15.5.2 of RFC9110}}), and a KYAPay token is not an `Authorization` credential.
A recipient MAY describe the reason in a problem details body {{RFC9457}} ({{DisclosingRequirements}}).

An agent that receives such a response can compare it with what it sent.
If it sent no token, or a token that was refused, it can obtain a new token and retry.
If the token lacked something the recipient requires, obtaining it may involve the human principal, for example, to agree to share an attribute or to complete further verification with the issuer, which can take minutes to days.

The same requirements can also be published in advance, so that an agent obtains a suitable token the first time.
This document does not define how.

## Account-Takeover Protectors {#ATOProt}

An ATO protector distinguishes legitimate account sessions from hijacked or unauthorized ones.
KYAPay tokens let it separate three cases that previously looked alike: a direct human-present session, a human-initiated agentic session, and an unauthorized automated session.

* It SHOULD treat a validated KYAPay token, bound to its request per {{TrustStack}}, as evidence that a session is a human-initiated agentic session rather than an unattended attack, and SHOULD record that distinction for auditing.

* When the agent needs to act within an existing account, the token MAY be exchanged for an OAuth access token using KYAPay Token Exchange {{I-D.skyfire-oauth-kyapay-token-exchange}} or OAuth 2.0 Token Exchange {{RFC8693}}.
A Security Token Service, Identity Provider, or OAuth 2.0 {{RFC6749}} Authorization Server validates the KYA token, extracts principal claims from `hid` (`hid.id`, and an email address or other identifier where the Target requested it), and issues an access token that the agent uses against the Target.

* The access token resulting from such an exchange SHOULD be sender-constrained (for example, via DPoP {{RFC9449}} or mutual TLS {{RFC8705}}) rather than issued as a freely transferable bearer token, so that exchanging the KYA token does not simply reintroduce a replayable credential.

* Cross-domain trust must be established: the Target's authorization server can fetch the issuer's keys via the `iss` JWKS endpoint, but it must also decide whether to trust that issuer and how to interpret its claims (see {{SecCon}}).

* A valid token attests authorization at the moment of issuance, not continuous human control.
  An ATO protector SHOULD NOT assume that the human remains in control for the token's whole lifetime, SHOULD prefer short token lifetimes, and SHOULD apply step-up ({{StepUp}}) for sensitive actions.

A KYAPay token attests the identity of the principal who authorized the agent.
It does not convey authority over any account at the Target.
An ATO protector MUST NOT treat a validated token, on its own, as evidence that the session is authorized for the account it is acting on.

Where a KYAPay token is presented to gain access to an account (at sign-in, at account creation, or when the token is exchanged for an access token as described above), a recipient SHOULD check that the token's principal is linked to that account.
The principal is linked when the account already records the token's `iss` and `hid.id`, or when a verified identifier in `hid` matches one the account records as verified.
A match is a strong positive signal that the agent is acting within its own principal's account, and a recipient MAY weight it accordingly.
A mismatch SHOULD be treated as grounds for step-up ({{StepUp}}) rather than as conclusive.
Because an issuer asserts an identifier only where it has verified that the principal controls it, a match is evidence of control rather than of mere knowledge.

## CIAM Systems

A CIAM system manages account creation and login for a Target's customers.
With KYAPay tokens it can manage agents consistently with humans:

* It MAY create an account for an agent (or for the human principal behind an agent) directly from a validated KYA or KYA-PAY token, drawing account attributes from the `hid` claim, such as an email address where the CIAM system requested it.

* It SHOULD use KYAPay Token Exchange {{I-D.skyfire-oauth-kyapay-token-exchange}} or OAuth 2.0 Token Exchange {{RFC8693}} to exchange a KYA token for an access token, enabling logged-in experiences (loyalty, saved preferences, offers) for human-via-agent sessions, applying the sender-constraining and cross-domain-trust considerations of {{ATOProt}}.

* It SHOULD maintain the linkage between a human principal and the agents authorized to act for them, so that a principal can be given visibility into, and control over, agent activity and vice versa.

* A validated token identifies the principal who authorized the agent; it does not by itself convey authority over an existing account.
  A CIAM system MAY create a new account from a validated token, in which case the token's principal is the account holder by construction, and it SHOULD record the token's `iss` and `hid.id` against the account.
  To admit an agent to an existing account, a CIAM system MUST additionally establish that the token's principal is linked to that account.
  Any of the following is sufficient: the account already records the token's `iss` and `hid.id`; a verified identifier in `hid` matches one the account records as verified; or an existing session for that account authenticated by other means.
  A mismatch, or the absence of any link, SHOULD be treated as grounds for step-up ({{StepUp}}) rather than as conclusive.
  A validated token alone MUST NOT be treated as sufficient.

## Consistency Across Layers

Because all of the intermediaries above validate the *same* token and draw on the *same* verified claims, an operator SHOULD configure them to reach consistent verdicts on a given request: a request admitted as a verified human-via-agent request by the bot manager should not be scored as an anonymous bot by the fraud manager.

## Granular Bad-Actor Mitigation and Triage {#BadActor}

The layered identity in a KYA token (`hid` for the principal, `apd` for the platform, `aid` for the agent instance) lets intermediaries choose the scope of a mitigation deliberately, rather than being forced into a single broad, "nuclear" block.
Because every request carries all three identities, a recipient can select the narrowest scope that is effective and escalate only as warranted:

* the individual **human principal** (`hid`) -- e.g., block or throttle one abusive user while all other users of the same agent continue to be served;

* the specific **agent instance** (`aid`) -- e.g., isolate one compromised or malfunctioning agent without affecting the principal's other agents or the rest of the platform; or

* the entire **agent platform** (`apd`) -- e.g., when an operator is systematically abusive, unresponsive, or untrusted, block or de-rate all traffic from that platform.

The same granularity supports responses short of blocking the requests, such as throttling, requiring step-up ({{StepUp}}), or lowering an assurance score, applied at whichever tier is appropriate.
Choosing the tightest effective scope isolates threats while protecting the reputation of well-behaved platforms and avoiding collateral damage to legitimate human-via-agent requests;
reserving platform-wide action for cases that genuinely warrant it keeps that option available without making it the default.

## Feedback to the Token Issuer {#IssuerFeedback}

The mitigation described above is local: a recipient acts on its own traffic.
Because the token issuer is the party that vouches for the initiator and can decline to vouch again, there is value in a feedback loop that lets a security intermediary report observed bad behavior back to the issuer, so that action can be taken at the source rather than only at each recipient independently.
When a security intermediary observes abusive or anomalous behavior that it can attribute, using the layered identity in the token, to a particular entity, it MAY report that observation to the token's issuer (identified by `iss`).
The attribution follows the same tiers as {{BadActor}}:

* the individual human principal (`hid`);
* the specific agent instance (`aid`); or
* the agent platform as a whole (`apd`).

On receiving such feedback, and subject to its own verification and anti-abuse safeguards, the issuer MAY take appropriate action, up to and including ceasing to issue tokens to the implicated initiator, agent, or platform -- the issuance-side counterpart to a recipient's refusal to accept a token, and complementary to the token-lifetime and revocation considerations of {{SecCon}}.
This mirrors the Certificate-Authority analogy of {{SecCon}}: just as a CA can stop issuing (and can revoke) certificates for a misbehaving subscriber, a KYAPay issuer can stop vouching for a misbehaving initiator.

This document does not define the mechanism, format, or trust model for this feedback channel;
how a recipient authenticates to an issuer, how reports are structured, how issuers guard against false or malicious reports, and what evidence is required are all open items.
Feedback SHOULD be treated as sensitive: reports identify principals, agents, or platforms and MUST be shared only with the relevant issuer and handled per {{PrivCon}}.
An issuer SHOULD corroborate feedback (for example, requiring reputation, multiple independent reports, or evidence) before taking action that would affect a principal, so that the channel cannot itself be used to deny service to legitimate initiators.

## Non-Payment and Handoff Flows

Not every human-via-agent flow includes a payment made by the agent.
In a common pattern, the agent performs discovery and assembles a cart but does not pay;
it then hands off to the human, who completes checkout in a browser, in a later session, or even in a physical store.
In such flows, the KYA token (carried without a PAY token) still provides the human presence signal that lets a security intermediary admit the agent's discovery and cart-building activity.
Intermediaries SHOULD accept KYA-only requests (those conveying a KYA token but no PAY token) for actions that do not themselves settle a payment, and MUST NOT treat the absence of a PAY token as a reason to classify an otherwise valid human-via-agent request as a bot.

Because `hid.id` is stable for the human principal at a Target service, for a given issuer and agent platform, a Target or its fraud manager MAY link the agent's earlier work at that service to the principal's eventual purchase there, for attribution, personalization and fraud analysis.
`hid.id`, `sub` and `aid.id` differ at every other Target service ({{I-D.skyfire-oauth-kyapay-token}}).
Linking to a purchase completed outside the agent session, or on a different channel, needs an identifier available in both places, such as an email address, which a token carries only where the Target requested it and the principal authorized its disclosure.

# Token Creation Considerations {#TokCreat}

This section is non-normative with respect to *how* tokens are created.
It records considerations that KYAPay Token Issuers and agent developers should weigh, but it deliberately does not mandate a mechanism.
The normative interoperability contract is the token defined by {{I-D.skyfire-oauth-kyapay-token}} and consumed as specified in {{SecInt}};
how a given agent obtains a valid token is an implementation and architecture choice.

## Why this Document is Not Prescriptive about Creation {#NonPrescCreat}

Readers accustomed to fixed API contracts may expect a specification to prescribe exactly how a credential is minted.
For agents, such prescription is neither possible nor desirable at this time, for several reasons:

* **Agent architectures are diverse.**
  Agents are built in fundamentally different ways -- computer-use agents that drive a desktop, browser-use agents that drive a browser, and programmatic agents that call APIs or tools -- and there is no single formula for building one.
  An agent may be an individual's personal, ad hoc creation, or a highly governed and deterministic enterprise agent for a regulated workflow, or anything in between.

* **Credentials may need to be shielded from the LLM.**
  In many designs, it is desirable that the LLM driving an agent never sees raw credentials.
  Token creation may therefore be placed inside deterministic tools that the LLM can invoke but whose outputs (the tokens, or the secrets used to mint them) it cannot read.
  Whether this matters depends on the deployment;
  a specification that mandated a single creation flow could not accommodate both shielded and unshielded designs.

* **Agent-identity and communication protocols are still emerging.**
  Multiple protocols for agent identity and for agent communication -- agent-to-tool (MCP {{MCP}}), agent-to-agent (A2A {{A2A}}), request signing ({{RFC9421}}), OAuth client identity ({{I-D.ietf-oauth-client-id-metadata-document}}), agent-to-API, Verifiable Intent {{VINTENT}}, UCP {{UCP}}, and others -- are under active development and require significant investment to reach production.
  Standardizing a single creation mechanism now would bind KYAPay to choices that are not yet settled.

* **The web already exists.**
  In contrast to those emerging interfaces, websites and merchant-controlled endpoints exist today at scale and require no new integration effort from merchants ({{DesignPrinc}}).
  A usable protocol must therefore work now, over the existing web, which argues for standardizing the delivered token rather than the creation path.

KYAPay's position is that the token is the fixed point, and creation is allowed to vary.
This is what lets a single token type serve individual and enterprise agents, shielded and unshielded designs, and today's web alongside tomorrow's agent protocols.

## Guidance for KYAPay Token Issuers

Given the above, KYAPay Token Issuers are encouraged to make token creation possible and easy regardless of the technology an agent uses:

* **Match the issuing interface to the agent.**
  A programmatic agent using a deterministic tool may be well served by a single REST endpoint with optional arguments -- one call that can mint a KYA, PAY, or KYA-PAY token with any supported settlement type (payment cards, stablecoins, and so on).
  An agent driven through MCP {{MCP}} may instead be better served by several narrow, specific tools (one per token type and per settlement type), so as to reduce the decision burden placed on the LLM.

* **Support multiple identity technologies.**
  Because there is no single agent-identity standard today, issuers should be prepared to accept and bind a variety of identity inputs when minting tokens, including (for example) CIMD {{I-D.ietf-oauth-client-id-metadata-document}}, WIMSE / SPIFFE {{SPIFFE}}, Agent Name Service (ANS), request-signing keys published in A2A Agent Cards {{A2A}}, and public/private key pairs.

* **Be available across agent technologies and protocols.**
  Issuers should be reachable by agents irrespective of the agent's runtime technology, the identity protocol it uses, and the communication protocol it speaks.

Origination and issuance need not be performed by the same party.
The `iss` claim identifies the party that signs (issues) the token, while the optional `ori` claim ({{I-D.skyfire-oauth-kyapay-token}}) identifies the party that originated -- that is, assembled and requested -- it;
these can be the same entity or different entities.
Likewise, the verified identity information carried in `hid` and `apd` may be supplied by the agent platform or a third-party verifier and merely attested by the issuer.
Verification itself need not be completed up front: it can be progressive, beginning with a lightweight check (such as email verification) and stepping up as a Target's requirements demand.
Consumers observe the result of these choices through the `verifier`/`verified` sub-claims and, where used, the
`ivm` {{I-D.skyfire-oauth-id-verification}},
`amr` {{I-D.skyfire-oauth-amr-values}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims, rather than through any mandated creation flow.

The objective of all of the above is a single outcome: a valid KYAPay JWT that is consumable by every transacting party -- the Initiator, the Target, and the bot managers, fraud managers, ATO protectors, and CIAM systems in between.

# Security Considerations {#SecCon}

When validating the JWTs described here and in {{I-D.skyfire-oauth-kyapay-token}}, implementers MUST follow the JSON Web Token Best Current Practices, BCP 225 {{RFC8725}}, in addition to the validation steps of {{I-D.skyfire-oauth-kyapay-token}}, Section 4.

## Transport Confidentiality

KYAPay tokens convey identity and, for PAY tokens, payment credentials.
They MUST be transmitted over TLS and MUST NOT be sent in the clear.

## Token Freshness and Lifetime

A long-lived token is a credential that can be captured, farmed, and resold for reuse against the same Target.
The `exp` claim MUST be present;
recipients SHOULD require short lifetimes (for high-assurance actions, on the order of a few minutes) and MUST reject tokens whose lifetime exceeds their local policy maximum.
Recipients SHOULD reject tokens whose `iat` is outside an acceptable window.

## Replay and Proof of Possession {#Replay}

A bare bearer token can be replayed by anyone who captures it within its validity window.
As deployed today ({{TrustStack}}), KYAPay accepts this bounded in-window risk and mitigates rather than eliminates it: recipients MUST confirm that the token is intended for them, as described in {{Counterparty}}, so that a token minted for one Target cannot be presented to another, and MUST keep accepted lifetimes short.
In-window replay within the token's audience is an accepted risk of the bearer model.
Because a conforming token is presented on many requests within its lifetime, a repeated `jti` is not by itself evidence of replay, and the claim is used for correlation, audit, and issuer feedback ({{IssuerFeedback}}) rather than for replay detection.
Where a deployment issues a distinct token for each request, a repeated `jti` does carry that meaning; this document describes the general case, in which it does not.
Where a token carries a proof-of-possession key (`cnf` {{RFC7800}}), recipients MUST additionally verify a per-request signature {{RFC9421}} over that key ({{PoPSignatures}}).
This stops a stolen token from being used on its own, turning an open-ended token-theft exposure into a bounded window in which a captured signed request could be replayed verbatim; it does not prevent that verbatim replay.
Access tokens produced by token exchange ({{ATOProt}}) SHOULD be sender-constrained ({{RFC9449}}, {{RFC8705}}).

## Malicious or Compromised Target Reuse

A Target that legitimately receives a bearer token can, within the token's window, reuse it to act elsewhere on the agent's behalf.
Audience binding limits this to the intended Target;
the proof-of-possession model ({{TrustStack}}), which binds each use to a fresh request signature, closes it.
Recipients and Targets MUST NOT forward received tokens to other parties (see {{PrivCon}}).

## Compromised Agent Host (Irreducible)

Malware resident on the agent's host can drive the agent's legitimate key as a signing oracle for as long as it is present.
Proof of possession does not prevent this: the malware's requests convey valid bindings and are indistinguishable from the real agent's.
This risk is bounded not by the token bindings but by short token lifetimes -- so anything signed during a compromise expires quickly once the malware is evicted -- together with host-level controls such as attestation, anomaly detection, and hygiene.

## Token Exposure in the Agent Runtime {#RuntimeExposure}

Agents driven by language models process untrusted content and record their activity in ways that conventional HTTP clients do not.
A KYAPay token can therefore leave the agent through normal operation, without the agent host being compromised.
Examples include:

* the token appearing in model context, conversation transcripts, or tool-call inputs and outputs that are logged, retained, or sent to a model provider;

* agent tracing, observability, or debugging systems that record outbound request headers;

* prompt injection, where content retrieved from a website or tool instructs the agent to reveal the token or to include it in a request to a party other than the intended Target.

For a bearer token ({{Bearer}}), anyone who obtains a leaked token can present it at the Target it was minted for until it expires.
Audience binding ({{Counterparty}}) does not help if the attacker presents the token at that Target.

Agent developers and KYAPay Token Issuers SHOULD mint and attach tokens in deterministic components ({{NonPrescCreat}}) whose outputs are not placed in model context.
Agents SHOULD attach a token only to requests sent to the Target for which the token was requested; where the token carries `tdm`, that means a destination within the domain that `tdm` names.
The component that attaches a token MUST make that decision without relying on model output.

Agent runtimes MUST NOT write KYAPay token values to logs, traces, or observability exports.

Agents SHOULD request a new token for each Target and task rather than reusing one broadly, and SHOULD request the shortest lifetime that fits the task.
Proof of possession ({{PoPPath}}) reduces this exposure only if the private key is also kept out of model context.

Some architectures place a token in model context unavoidably.
Where an agent obtains a token by invoking a tool, the tool result is returned to the model, and any retained transcript of that session therefore contains the token.
A deployment in that position SHOULD treat the token as disclosed to every party with access to that context, including the model provider.
For such a deployment, per-Target, short-lived tokens are the main mitigation, together with proof of possession where the private key is held outside model context.

## Issuer Trust

A token is only as trustworthy as its issuer and the verifiers it cites.
The system is federated: many issuers are possible, and a recipient MUST maintain an explicit set of trusted issuers and the claims and assurance levels it will accept from each.
Establishing this trust at scale is an open problem analogous to the Certificate Authority model (audits, a maintained issuer list, a removal mechanism, and possibly transparency logs);
this document does not define such a framework, and deployments should not assume one exists.
Until it does, trust is established out of band (for example, configured relationships with well-known issuers).

## Revocation

An agent whose platform is deregistered, or a human whose identity verification is revoked, may still hold unexpired tokens.
This specification does not define a revocation mechanism.
Recipients SHOULD keep accepted lifetimes short to bound exposure, and SHOULD consult an issuer's live-status or revocation endpoint, where one is offered, for high-assurance or high-value actions.

## Continuity of Human Control

A valid token attests that the human principal authorized the agent at the time of issuance.
It does not prove the human remains in control -- for example, if the agent is later prompt-injected, or if credentials are exfiltrated from the agent or its host.
Recipients SHOULD treat high-value actions as warranting step-up ({{StepUp}}) rather than relying solely on a previously issued token.

## Key Management

If an issuer or platform rotates a signing key, tokens signed with the old key may fail validation prematurely.
Issuers SHOULD retain superseded verification keys in their JWK Set for at least the maximum token lifetime after rotation.

## Disclosing Requirements {#DisclosingRequirements}

The `Accept-KYAPay-Token` field ({{AcceptField}}) and any problem details a recipient returns ({{StepUp}}) tell the sender what the recipient requires.
That helps a legitimate agent obtain a suitable token, and it also reaches anyone else who sends a request.
How much to state is a decision for the recipient, in light of its own risk:

* It can omit the field, state only some of its requirements, or leave out the `issuers` parameter, which names the issuers it accepts.

* It can instead publish its requirements in advance, or agree them with initiators out of band.

* Requirements that differ by principal, by account, or by the outcome of a risk assessment can reveal those facts, for example, which accounts are of high value or that a request has been flagged ({{Section 9 of RFC9470}} describes the same risk for authentication requirements).

* Saying which validation check a token failed can help an attacker probe the recipient; a generic reason avoids this ({{Section 5 of RFC9457}}).

Stating requirements does not weaken the token itself: a request still succeeds only with a token signed by an issuer the recipient trusts.

# Privacy Considerations {#PrivCon}

The privacy considerations of {{I-D.skyfire-oauth-kyapay-token}} apply.
In particular:

## Minimal Disclosure

Only the information needed to facilitate the intended interaction should be placed in a token and conveyed to intermediaries and Targets.
Issuers and senders SHOULD prefer the least revealing set of claims sufficient for the decision at hand -- for example, an assurance level or a boolean verified status rather than underlying identity documents or personally identifying verification data.
For example, a bot manager deciding whether to admit a request to browse a catalog needs no identifying claim: `sub` and `hid.id` let it rate-limit and recognize a returning principal at that Target service, and the assurance claims in `hid` say how strongly the principal was verified.
By default, a token identifies the principal only by `hid.id` ({{I-D.skyfire-oauth-kyapay-token}}); identifying claims are added only where a Target requires them for the action at hand.
Because KYA tokens are JWTs, selective disclosure (for example, SD-JWT {{RFC9901}}) is a forward-compatible path to revealing only the claims a given Target needs.

## Consent

Tokens assert that an agent acts on behalf of a principal.
That assertion is only legitimate when the principal has authorized the interaction.
Systems SHOULD be designed so that tokens are minted only with the principal's authorization.

## Handling by Intermediaries

Security intermediaries receive verified personal data in the course of making decisions.
They MUST treat `KYAPay-Token` values and their decoded contents as sensitive personal data: not caching them for replay, not logging them in the clear, and not sharing them with parties other than legitimate participants in the interaction.

# IANA Considerations

## HTTP Field Name Registration

IANA is requested to register the following entries in the "Hypertext Transfer Protocol (HTTP) Field Name" registry established by {{RFC9110}}:

* Field Name: KYAPay-Token
* Status: permanent
* Structured Type: List
* Reference: {{HeaderField}} of this document
* Comments: Carries one or more KYAPay tokens in an HTTP request
^
* Field Name: Accept-KYAPay-Token
* Status: permanent
* Structured Type: Dictionary
* Reference: {{AcceptField}} of this document
* Comments: States what a request must carry for a KYAPay recipient to accept it

## JWT and Media Type Registrations

This document defines no new JWT claims, JWT confirmation methods, or media types.
The claims and media types used by KYAPay tokens are registered by {{I-D.skyfire-oauth-kyapay-token}}.
The `amr` extensions and the `ivm` and `aml` claims and values are registered by
{{I-D.skyfire-oauth-amr-values}},
{{I-D.skyfire-oauth-id-verification}}, and
{{I-D.skyfire-oauth-aml-methods}}, respectively.

--- back

# Acknowledgments
{:numbered="false"}

The authors thank the contributors to the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}} specification and the partners in the KYAPay consortium (see {{KYAPAY-ORG}}) -- including bot-management, fraud, CIAM, and ATO vendors and merchants -- whose deployment and review experience informed the usage patterns and open issues described here.

We would like to thank
Bhasker Allam,
Jean Diaconu,
and
David Senecal
for their contributions to the specification.
Bhasker Allam also wrote the text on which {{RuntimeExposure}} is based.


# Document History
{: numbered="false"}

[[ to be removed by the RFC Editor before publication as an RFC ]]

-02

* Updated the traditional bot detection methods in the introduction
* Updated the Bot Managers section and introduce the concept of feedback loop to the agent through an error response in case of token assessment failure
* Stated that the principal need not be present at the time of a human-via-agent request.
* Made admission depend on the site's policy for the requested action, and stated that rate and abuse limits and step-up still apply to admitted requests.
* Stated that a token is reusable within its validity period and audience, and that a repeated `jti` is not by itself evidence of replay, and rewrote the replay considerations to match.
* Described how a recipient confirms the audience: `aud` and `tsi` where it has onboarded to the issuer, `tdm` otherwise; required `tdm` where the Target is identified by a domain name.
* Prohibited retrieving key material for an issuer that is not trusted.
* Made BCP 225 a normative reference, and let a recipient reject a request carrying more than one token of the same `typ`.
* Added Token Exposure in the Agent Runtime to the Security Considerations.
* Added Request Signatures for Proof of Possession: the covered components, and verification at the first hop.
* Added Attaching Tokens to Requests, scoped the zero-tech-lift principle to agents that control a request's header fields, and added CORS preflight and caching requirements.
* Described retaining a token for audit or dispute resolution.
* Defined `KYAPay-Token` as a List of Tokens, replacing the ABNF, and required a recipient to reject a field that fails to parse.
* Added the `Accept-KYAPay-Token` response field, which a recipient MAY send, replaced Step-Up Authentication with Responding When a Token Falls Short (over HTTP, 403, or 402 for payment, unless the protocol in use specifies its own errors; not 401), changed the Bot Managers error examples to match, made a recipient decide from its own policy whether it uses tokens for a request and decline, within its role, a request whose token fails validation, rather than treat it as carrying no token, and added Disclosing Requirements.
* Used `hid.id` for account binding at ATO protectors and CIAM systems, defined the base claims for hosts with no claims set, made linkage per Target service, and added a minimal-disclosure example.

-01

* Rewrote the Introduction to frame agents as an evolution of verified bot traffic.
* Clarified the meaning of Verifier and differentiated from Recipient.
* Added {:vspace} syntax to definition list entries.

-00

* Initial draft.
