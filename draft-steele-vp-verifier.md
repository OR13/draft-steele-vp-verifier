---
title: "VP Verifier"
category: info

docname: draft-steele-vp-verifier-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - verifiable credentials
 - decentralized identifiers
 - http apis
venue:
  github: "OR13/draft-steele-vp-verifier"
  latest: "https://OR13.github.io/draft-steele-vp-verifier/draft-steele-vp-verifier.html"

author:
 -
    fullname: "Orie Steele"
    organization: Transmute
    email: "orie@transmute.industries"

normative:
  RFC7518:
  RFC7519:
  RFC7800:
  RFC9449:
  RFC2397:
  W3C-VC:
    title: "Verifiable Credentials Data Model v2.0"
    target: https://www.w3.org/TR/vc-data-model-2.0/
  Secure-VC:
    title: "Securing Verifiable Credentials using JOSE and COSE"
    target: https://www.w3.org/TR/vc-jose-cose/

informative:
  OpenID4VP:
    author:
      org: "OpenID Foundation"
    title: "OpenID for Verifiable Presentations"
    target: "https://openid.net/specs/openid-4-verifiable-presentations-1_0.html"

---

--- abstract

This document describes a protocol for exchanging Verifiable Presentations using HTTP.

--- middle

# Introduction


The Verifiable Credentials (VC) data model is described in {{W3C-VC}}, and securing mechanisms based on JOSE and COSE are described in {{Secure-VC}}.

Verifiable Presentations (VP) are described in both {{W3C-VC}} and {{Secure-VC}}.

Although HTTP based protocols exist for making Verifiable Presentations, such as {{OpenID4VP}}, they are encumbered by the conventions and history of OAuth and the experience needed to deploy OpenID successfully.

This document describes an HTTP API for VP Verifiers, that provides interoperability while limiting normative dependencies, optionality and excessive cryptographic agility.

The API this document specifies is designed with priority for use by organizations or businesses that need to exchange presentations with other organizations or governments.

Although this API describes HTTP resources which require authentication, a specific authentication mechanism is not specified.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


The terms issuer, subject and holder are defined in {{W3C-VC}}.

The terms `iat`, `exp`, `iss`, `sub`, `aud`, are defined in {{RFC7519}}.

The term `nonce` is defined in {{RFC9449}}.

The term `cnf` is defined in {{RFC7800}}.

The term `ES256` is a digital signature algorithm described in {{Section 3.4 of RFC7518}}.

This document uses "..." to ellide text in examples for readability.


# Mandatory to Implement

Although {{Secure-VC}} describes several different media types, and each media type can be secured with many different signing or encryption algorithms, this documents specifies the following as mandatory to implement:

The following media types MUST be supported as JWT Claim Sets in the request body of the presentation API:

- application/vp+jwt

The following media types MUST be supported as Enveloped Credentials in presentation JWT Claim Set:

- application/vc+jwt

The following JSON Web Signature algorithms MUST be supported:

- ES256

This document might be updated in the future to support additional media types or signing algorithms.

# Verifier Resources

## Presentations

The `/presentations` resource describes the collection of verifiable presentations associated with a verifier.

This collection supports creating resources.

### Create

Creating a verifiable presentation requires several individual steps to be completed.

The requirements for implementing Verifiable Presentations as described in {{W3C-VC}} and {{Secure-VC}} in ways that support both credential confirmation and replay attack protection are not well defined at the time this document was written.

Not all credentials will support confirmation, but all presentations can be made to mitigate replay attacks.

The following text describes a simple flow that cannot support confirmation or mitigate replay attacks, which is improved upon in the subsequent section.

First the holder needs to gather all the verifiable credentials that need to be presented, and to construct a Verifiable Presentation that includes them.

Note that each verifiable credential in application/vc+jwt format is encoded as a data URI as described in {{RFC2397}}.

Next the holder signs the presentation to produce a JWT of content type application/vp+jwt.

The following informative example displays a decoded verifiable presentation:

~~~
{
  "typ": "vp+jwt",
  "alg": "ES256",
  "kid": "https://example.gov/keys/42"
}.{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
  ],
  "type": "VerifiablePresentation",
  "holder": "https://example.gov",
  "verifiableCredential": [{
    "@context": ["https://www.w3.org/ns/credentials/v2"],
    "type": ["EnvelopedVerifiableCredential"],
    "id": "data:application/vc+jwt,eyJraW..."
  }]
}
~~~
{: #example-decoded-verifiable-presentation align="left" title="A Verifiable Presenation"}

Next the holder uses an http client to submit this verifiable presentation to a verifier:

~~~
Request:

POST /presentations HTTP/1.1
Host: example.gov
Authorization: Bearer ey...
Content-Type: application/vp+jwt
Accept: application/json

eyJhbGciOiJFZE...

Response:

HTTP/1.1 200 OK
Content-Type: application/json

{
  ...
}

~~~
{: #example-presentation-request align="left" title="Submitting a Verifiable Presentation"}

The server MUST authenticate the client submitting the presentation.
The server SHOULD verify the presentation asynchronously, but MAY return verification status in response to a presentation submission.

The submission of presentation tokens and enveloped verifiable credentials is subject to replay attacks.
The following section is dedicated to discussing the submission of presentations with replay protection and of credentials with confirmation methods.

#### Interactive Presentations

Verifiers who wish to be assured that a holder making a presentation still controls a specific cryptographic key, or who wish to receive presentations of credentials that include confirmation need to support interactive presentation submission.

An interactive presentation starts with a holder requesting a `nonce` from the verifier:

~~~

Request:

POST /nonce HTTP/1.1
Host: example.gov
Content-Length: 0
Authorization: Bearer ey...
Accept: application/json

Response:

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "c_nonce": "wKI4LT17ac15ES9bw8ac4",
  "c_nonce_expires_in": 120
}

~~~
{: #example-nonce-request align="left" title="Requesting a Nonce"}

The holder then signs this nonce, to produce a confirmation token:

~~~
{
  "typ": "...+jwt",
  "alg": "ES256",
  "kid": "..."
}.{
  "aud": "https://example.gov",
  "iat": 1701960444,
  "nonce": "LarRGSbmUPYtRYO6BQ4yn8"
}
~~~
{: #example-confirmation-jwt align="left" title="Confirmation Token"}

The example above is informative.
The structure of a confirmation token is credential type specific.

This process is repeated for each credential that contains a confirmation method, using the key or identifier the confirmation method specifies.
Each confirmation token is then embedded in a Verifiable Presentation next to the credential that requires it.

Finally the holder then signs the nonce and audience, as part of their production of the application/vp+jwt:

For example:

~~~
{
  "typ": "vp+jwt",
  "alg": "ES256",
  "kid": "https://example.gov/keys/42"
}.{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
  ],
  "type": "VerifiablePresentation",
  "holder": "https://holder.example.gov",
  "aud": "https://verifier.example.gov",
  "nonce": "LarRGSbmUPYtRYO6BQ4yn8",
  "verifiableCredential": [{
    "@context": ["https://www.w3.org/ns/credentials/v2"],
    "type": ["EnvelopedVerifiableCredential"],
    "id": "data:application/vc+jwt,eyJraW..."
  },{
    "@context": ["https://www.w3.org/ns/credentials/v2"],
    "id": "data:application/...+jwt,eyJraW..."
  }]
}
~~~
{: #example-decoded-verifiable-presentation-with-nonce align="left" title="A Verifiable Presenation for an Audience"}

In the example above, we assume that the second element of "verifiableCredential" contains a data uri that includes the confirmation token for the credential in the first element.

This structure allows verifiers who usually process confirmation tokens along with credentials in JWT format, to receive both the credential and the confirmation token in a single signed presentation request.

In the case that additional status information needs to be presented by the holder to the verifier, such as a status list or status assertion, those credentials can be presented along with the other credentials embedded in the signed presentation.

It is worth noting that some credential formats such as application/vc+sd-jwt or application/vc+cose can transport confirmation tokens alongside or inside the original credential.

Future revisions to this document might describe how to submit verifiable presentations that rely on structured suffixes other than +jwt.

# Security Considerations

## Private Keys

It is important to protect the signing key used to issue presentations, as well as any confirmation keys associated with the credentials.
It is RECOMMENDED that all private keys be initialized such that they cannot be exported.

## Authorization

HTTP Clients that submit verifiable presentations MUST be authenticated.
A client may submit credentials related to or bound to many different subjects.
A full description of policy for verifying presentations from holders is beyond the scope of this document.

## HTTPS

HTTPS MUST be used with all resources described in this document.
Verifiers MUST support at least TLS 1.3 or a more recent version.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
