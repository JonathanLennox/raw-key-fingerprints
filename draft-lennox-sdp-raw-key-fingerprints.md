---
title: "Session Description Protocol Fingerprints for Raw Public Keys in (Datagram) Transport Layer Security"
abbrev: "SDP Fingerprints for Raw Keys in (D)TLS"
category: std

docname: draft-lennox-sdp-raw-key-fingerprints-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "ART"
keyword:
 - sdp
 - fingerprints
 - raw public keys
venue:
  github: "JonathanLennox/raw-key-fingerprints"

author:
 -
    fullname: Jonathan Lennox
    organization: 8x8, Inc / Jitsi
    email: jonathan.lennox@8x8.com

normative:

informative:


--- abstract

This document defines how to negotiate the use of raw keys for TLS and DTLS
with the Session Description Protocol (SDP). Raw keys are more efficient
than certificates for typical uses of TLS and DTLS negotiated with
SDP, without loss of security.

--- middle

# Introduction

When a Transport-Layer Security (TLS) {{!RFC8446}} {{!RFC5246}} or Datagram
Transport-Layer Security (DTLS) {{!RFC9147}} {{!RFC6347}}
connection is negotiated using the Session Description Protocol {{!RFC8866}},
certificates are validated using certificate fingerprints specified in
the SDP {{!RFC8122}}, rather than by any information carried in the certificate.
Typically these certificates are self-signed.
The only information carried in these certificates that is used by the
process are the public keys; the rest of the information is useless.
This other information can be large, and once post-quantum
public keys are needed, the self-signed signature in particular will
be very large.

TLS and DTLS now
support using raw keys, rather than X.509 certificates, in
circumstances such as these {{!RFC7250}}.  This document defines how such raw key
certificates can be negotiated in SDP.

## Certificate overheads

As an illustration of the overheads of self-signed certificates, the
self-signed certificate in a WebRTC DTLS handshake negotiated by a
recent version of Chrome is 282 bytes, of which only 91 bytes are the
public key (the subjectPublicKeyInfo in id-ecPublicKey format
{{?RFC5480}}).

For the transition to post-quantum keys and signatures, the overhead
is significantly worse.  For ML-DSA level 3 certificate, described in {{Appendix B of
?I-D.ietf-lamps-dilithium-certificates}}, public keys are 1952
bytes, and signatures are 3293 bytes.  Only the public key is useful --
the signature (along with any other certificate overhead) is wasted
space.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol

## The "raw-key-fingerprint" attribute. {#sdp-attribute}

This document defines an SDP attribute, "raw-key-fingerprint".
Its syntax is the same as the "fingerprint" attribute defined in
{{!RFC8122}}, as specified in {{figabnf}}.

~~~~~~~~~~

attribute /= raw-key-fingerprint-attribute

raw-key-fingerprint-attribute = \
                      "raw-key-fingerprint" ":" hash-func fingerprint
                      ; hash-func and fingerprint are defined
                      ; in [RFC8122]
~~~~~~~~~~
{: #figabnf title="ABNF for the raw-key-fingerprint attribute"}

A raw key fingerprint is a secure one-way hash of the distinguished
Encoding Rules (DER) form of the subjectPublicKeyInfo (SPKI) of the
raw key, in the form specified in {{Section 3 of !RFC7250}} when the
appropriate certificate_type value is RawPublicKey.

> The subjectPublicKeyInfo structure encodes the key type as well as
> its value, so the meaning of the key is unambiguous.

The one-way hash is performed using the hash function specified in the
'hash-func' field, which MUST be a hash function from the IANA table
"Hash Function Textual Names" defined in {{!RFC8122}}, with 'SHA-256'
preferred.  As in {{!RFC8122}}, the hash functions 'MD2' and 'MD5'
MUST NOT be used either for generating or verifying raw key fingerprints.

Also as in {{!RFC8122}}, the raw key fingerprint is represented as
upper-case hexadecimal bytes, separated by colons.  The number of
bytes is defined by the hash function.



## SDP Offer/Answer procedures

In SDP offer/answer {{!RFC3264}}, if the endpoint creating an SDP
offer wishes to use raw public keys for TLS or DTLS, the offerer
includes an SDP "raw-key-fingerprint" attribute describing its raw
public key at the session level or the appropriate media levels.  In
an initial offer, it SHOULD also include a valid SDP "fingerprint"
attribute for a self-signed X.509 certificate as defined in
{{!RFC8122}}, unless it knows for certain through out-of-band means
that the peer that will be performing the answer definitely supports
raw keys.

In its answer, the answerer then includes a "raw-key-fingerprint" with
the fingerprint of its own raw public key.  It MAY omit the SDP
"fingerprint" attribute.

Either the offerer or the answerer MAY include multiple
"raw-key-fingerprint" attributes, for example if they want to provide
a fingerprint hashed with multiple different hash functions, or if the
media negotiated by the offer/answer might end up at one of several
different endpoints which have different public keys.

In subsequent offers, an offerer MUST send the
"raw-key-fingerprint" value of the raw public key that it used in the
TLS/DTLS session as long as that TLS/DTLS session
remains.  It MAY omit "fingerprint" attributes, and
"raw-key-fingerprint" attributes for unused raw public keys,
when the state of the connection attribute {{!RFC4145}} is "existing"
and the value of the raw key fingerprint is unchanged.
If it sends an offer with "connection:new", or the fingerprint
changes, it SHOULD include both "fingerprint" and
"raw-key-fingerprint" attributes under the same rules as it would use
for an initial offer.

### TLS/DTLS procedures for SDP Offer/Answer connections.

The TLS client and server roles are negotiated for the session
following the mechanisms defined in {{!RFC4145}}; the endpoint in the
"active" role will be the client.

If raw keys have been offered in SDP, the initial ClientHello of the
transaction MUST include both a ClientCertTypeExtension and a
ServerCertTypeExtension including RawPublicKey as one of the types.
If the client has already seen its peer's offer or answer including a
"raw-key-fingerprint" SDP attribute, this MAY be the only type listed
in the extensions; otherwise, if the client's offer or answer included
a "fingerprint" attribute, the extension lists MUST also include X509.

The server's ServerHello MUST then send a ClientCertTypeExtension
and a ServerCertTypeExtension listing RawPublicKey as the type, as
well as its own raw public key in the Certificate and a certificate
request for the client.  The client then sends its own raw key.

Both client and server MUST verify that the raw key fingerprint
signaled in SDP matches that of the raw public key received over TLS
or DTLS,
and terminate the TLS or DTLS connection with a bad_certificate error
if it does not.  Note that in some circumstances a ClientHello or ServerHello
may arrive before an SDP answer; application data MUST NOT be sent over the
TLS connection until the raw key fingerprint has been received and
validated.

If multiple raw key fingerprints are present, a certificate is valid
if at least one raw key fingerprint matches the raw key using a hash
function that the entity considers sufficiently secure.

If a remote SDP offer or answer does not contain a "raw-key-fingerprint"
attribute, or TLS does not contain a CertTypeExtension containing
RawPublicKey, then the peer
does not support (or does not wish to use) raw keys in fingerprints,
and the standard procedures of {{!RFC8122}} MUST be used instead.

If an SDP offer or answer is inconsistent with TLS messaging (e.g., a
CertType of X509 is provided when SDP contained only
"raw-key-fingerprint" attributes, or a CertType of RawPublicKey when SDP
contained only "fingerprint" attributes), then the TLS connection MUST
be terminated with a bad_certificate error.

> TODO: Do we want to support asymmetric cases, where one endpoint is
> willing to receive raw keys but cannot present one?  This would be
> relatively straightforward for answerers, but would require an
> additional SDP attribute for an offerer to indicate its willingness
> to receive a raw key.

## SDP Advertisements

Older uses of SDP (such as RTSP {{?RFC7826}}) use advertised SDP
rather than offer/answer.  In this mode, an entity presents an SDP
description that other endpoints can then connect to freely, without
providing a matching SDP description.

When raw key fingerprints are used in this case, the SDP describes the
TLS server.  Clients connect using the standard TLS client/server
procdure. Clients MUST validate that the raw key provided in the
connection matches the raw key fingerprint in the SDP.

# Possible follow-ons {#follow-ons}

Two protocols have defined mechanisms by which SDP fingerprints can be
signed to ensure their end-to-end security: PASSPorT {{?RFC8225}} and
WebRTC Identity Providers {{?RFC8827}}.  Unfortunately, the latter has
seen no deployment as far as the author is aware; and the former,
while widely implemented to authenticate calling party telephone
numbers, has not seen much if any adoption of the mode that signs
certificate fingerprints.

Nonetheless, both of these mechanisms could easily be extended so as
to secure raw key fingerprints as well.

> TODO: Should we actually define these extensions?  Is it worth the
> trouble?

# Design alternatives

As an alternative to a new "a=raw-key-fingerprint" attribute, an
alternative signaling mechanism would be to define new values for the
"hash-func" field of the existing "a=fingerprint" attribute, e.g. the
"hash-func" value "raw-key-SHA-256" in an "a=fingerprint" attribute
would mean to use the mechanisms defined in this document.

This would have the advantage of allowing existing mechanisms that use
and transport "a=fingerprint" values (both existing code, and
mechanisms such as those discussed in {{follow-ons}}) to support the
raw key mechanism, without requiring new code to be written or new
extension points to be defined.

This would be at the cost of somewhat less clean signaling; and in
particular, the definition of the "Hash function textual names"
IANA registry would get complicated.

This would require discussion during the consensus process as this
mechanism is standardized.

# Security Considerations

The security of a TLS or DTLS connection negotiated using the
mechanisms defined in this document is identical to that of a
connection negotiated via the mechanism in {{!RFC8122}}.  All the
security considerations of that document also apply to this one.  As
identity information is normally not used from the SDP-negotiated
certificates, this mechanism should have identical security properties
to that of {{!RFC8122}}.

As with {{!RFC8122}} fingerprints, the mechanism in this document is
vulnerable to an attacker who can modify the signaled fingerprints and
launch a meddler-in-the-middle attack.  See {{follow-ons}} for various
proposed methods to prevent this attack.


# IANA Considerations

This document defines an SDP session and media-level attribute:
'raw-key-fingerprint'.  Its format is defined in Section
{{sdp-attribute}}.  This attribute should registered by IANA under the
"att-field (both session and media level)" registry within the
"Session Description Protocol (SDP) Parameters" registry.



--- back

# Acknowledgments
{:numbered="false"}

Thanks to Martin Thomson, Harald Alvestrand, and Mike Ounsworth for their comments on
this document.
