---
title: "Service Binding Mapping for DNS URIs"
abbrev: "SVCB for dns://"
docname: draft-schwartz-svcb-dns-latest
category: info

ipr: trust200902
area: General
workgroup: dnsop
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Schwartz
    name: Benjamin Schwartz
    organization: Google LLC
    email: bemasc@google.com

normative:
  RFC2119:

informative:



--- abstract

The SVCB DNS record type expresses a bound collection of endpoint metadata, for use when establishing a connection to a named service.  DNS itself can be such a service, when the server is identified by a hostname in a `dns:` URI.  This document provides the SVCB mapping for named DNS servers, allowing DNS servers to indicate support for new transport protocols.

--- middle

# Introduction

The SVCB record type {{!SVCB=I-D.draft-ietf-dnsop-svcb-https-01}} provides clients with information about how to reach alternative endpoints for a service, which may have improved performance or privacy properties.  The service is typically identified by its authority (a hostname and optionally a port) and scheme (typically a URI scheme).

The `dns:` URI scheme {{!DNSURI=RFC4501}} describes a way to represent DNS queries as URIs.  This scheme optionally includes an authority, comprised of a host and port number (with a default of 53).  DNS URIs often omit the authority, or specify an IP address, but a hostname is also a supported authority.

Use of the SVCB record type with a URI scheme requires a mapping document, indicating how a client for that scheme can interpret the contents of the SVCB SvcParams.  This document provides the mapping for DNS URIs that contain a hostname authority, allowing the server to offer alternative endpoints and transports, including encrypted transports like DNS over TLS and DNS over HTTPS.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Name form

Names are formed using Port-Prefix Naming ({{SVCB}} Section 2.3).  For example, `dns://dns1.example.com:5353` would be converted to the domain `_5353._dns.dns1.example.com.`.

# Applicable existing SvcParamKeys

## port

This key is used to indicate the target port for connection.  If omitted, the client SHALL use the default port for each transport protocol: 853 for DNS over TLS and 443 for DNS over HTTPS.

This key is automatically mandatory if present.

## alpn and no-default-alpn

These keys indicate the set of supported protocols.  The default protocol is "dot", indicating support for DNS over TLS {{!DOT=RFC7858}}.

If the protocol set contains any HTTP versions (e.g. "h2", "h3"), then the record indicates support for DNS over HTTPS {{!DOH=RFC8484}}, and the "dohpath" key MUST be present.  All keys specified for use with the HTTPS record are also permissible, and apply to the resulting HTTP connection.

If the protocol set contains protocols with different default ports, and no port key is specified, then protocols are contacted separately on their default ports.  Note that in this configuration, ALPN negotiation does not defend against cross-protocol downgrade attacks.

These keys are automatically mandatory if present.

## echconfig

The echconfig SvcParamKey, if present, applies to any ECH-capable connection that uses this record.

# New SvcParamKeys

## dohpath

"dohpath" is a single-valued SvcParamKey whose value (both in presentation and wire format) is a relative URI Template {{!RFC6570}}, normally starting with `/`.  If the "alpn" SvcParamKey indicates support for HTTP, clients MAY construct a DNS over HTTPS URI Template by combining the prefix "https://", the authority hostname from the `dns://` URI, the port from the "port" key if present, and the dohpath value.  (The port from the `dns://` URI MUST NOT be used.)

Clients SHOULD NOT query for any "HTTPS" RRs when using the constructed URI Template.  Instead, the SvcParams and address records associated with this SVCB record SHOULD be used for the HTTPS connection, with the same semantics as an HTTPS RR.  However, for consistency, server operators SHOULD publish an equivalent HTTPS RR, especially if clients might learn this URI Template through a different channel.

# Limitations

DNS URIs convey limited information to the client.  For example, they do not indicate whether the query should include the "recursion desired", "DNSSEC OK", or "checking disabled" flags.  Clients must know the appropriate values for these flags in their use case.  Similarly, nothing in the DNS URI or in this document indicates the set of names for which the server is willing to answer queries.

# Examples

* A resolver at `dns://resolver.example` that supports
  * DNS over TLS on `resolver.example`, port 853 and 8530, with `resolver.example` as the Authentication Domain Name,
  * DNS over HTTPS at `https://resolver.example/dns-query{?dns}`, and
  * an experimental protocol on a different endpoint:

        $ORIGIN example.
        _dns.resolver 7200 IN SVCB 1 resolver (
          alpn=h2,h3 echconfig=... dohpath=/dns-query{?dns} )
        _dns.resolver 7200 IN SVCB 2 resolver (
          port=8530 echconfig=... )
        _dns.resolver 7200 IN SVCB 3 fooexp.resolver ( port=5353
          echconfig=... alpn=foo no-default-alpn foo-info=... )

* A nameserver at `dns://ns.example` whose service configuration is published on a different domain:

      $ORIGIN example.
      _dns.ns 7200 IN SVCB 0 _dns.ns.nic

# Security Considerations

Clients MUST authenticate the server to its name during secure transport establishment.  This name is the hostname present in the DNS URI, and cannot be influenced by the SVCB record contents.  Accordingly, this draft does not mandate the use of DNSSEC.  This draft also does not specify how clients authenticate the name (e.g. selection of roots of trust), which might vary according to the context.

A client that attempts a connection using an encrypted DNS transport from a SVCB record SHOULD NOT fall back to unencrypted DNS if connection fails.  (This is different from the advice in Section 3 of {{SVCB}}, which assumes the default transport is secured.)  Specifications making using of this mapping MAY adjust this fallback behavior to suit their requirements.

# IANA Considerations

IANA would be directed to register the "dohpath" key in the SVCB Service Parameters registry.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.