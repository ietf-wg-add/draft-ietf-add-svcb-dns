---
title: "Service Binding Mapping for DNS Servers"
abbrev: "SVCB for DNS"
docname: draft-schwartz-svcb-dns-latest
category: std

ipr: trust200902
area: General
workgroup: add
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

The SVCB DNS record type expresses a bound collection of endpoint metadata, for use when establishing a connection to a named service.  DNS itself can be such a service, when the server is identified by a domain name.  This document provides the SVCB mapping for named DNS servers, allowing them to indicate support for new transport protocols.

--- middle

# Introduction

The SVCB record type {{!SVCB=I-D.draft-ietf-dnsop-svcb-https-02}} provides clients with information about how to reach alternative endpoints for a service, which may have improved performance or privacy properties.  The service is identified by a "scheme" indicating the service type, a hostname, and optionally other information such as a port number.  A DNS server is often identified only by its IP address (e.g. in DHCP), but in some contexts it can also be identified by a hostname (e.g. "NS" records, manual resolver configuration) and sometimes also a non-default port number.

Use of the SVCB record type requires a mapping document for each service type, indicating how a client for that service can interpret the contents of the SVCB SvcParams.  This document provides the mapping for the "dns" service type, allowing DNS servers to offer alternative endpoints and transports, including encrypted transports like DNS over TLS and DNS over HTTPS.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Name form

Names are formed using Port-Prefix Naming ({{SVCB}} Section 2.3).  For example, a DNS service identified by the name "dns1.example.com" and (unusually) the non-default port number 5353 would be represented as `_5353._dns.dns1.example.com.`. A DNS service using the default port number of 53 would be represented as `_dns.dns1.example.com.`.

# Applicable existing SvcParamKeys

## alpn and no-default-alpn

These keys indicate the set of supported protocols ({{SVCB}} Section 6.1).  The default protocol is "dot", indicating support for DNS over TLS {{!DOT=RFC7858}}.

If the protocol set contains any HTTP versions (e.g. "h2", "h3"), then the record indicates support for DNS over HTTPS {{!DOH=RFC8484}}, and the "dohpath" key MUST be present ({{dohpath}}).  All keys specified for use with the HTTPS record are also permissible, and apply to the resulting HTTP connection.

If the protocol set contains protocols with different default ports, and no port key is specified, then protocols are contacted separately on their default ports.  Note that in this configuration, ALPN negotiation does not defend against cross-protocol downgrade attacks.

These keys are automatically mandatory if present.  (See Section 7 of {{SVCB}} for the definition of "automatically mandatory".)

## port

This key is used to indicate the target port for connection (({{SVCB}} Section 6.2)).  If omitted, the client SHALL use the default port for each transport protocol (853 for DNS over TLS, 443 for DNS over HTTPS).

This key is automatically mandatory if present.

## Other applicable SvcParamKeys

These SvcParamKeys apply to the "dns" scheme without modification:

* echconfig
* ipv4hint
* ipv6hint

# New SvcParamKeys

## dohpath {#dohpath}

"dohpath" is a single-valued SvcParamKey whose value (both in presentation and wire format) is a relative URI Template {{!RFC6570}}, normally starting with "/".  If the "alpn" SvcParamKey indicates support for HTTP, clients MAY construct a DNS over HTTPS URI Template by combining the prefix "https://", the service name, the port from the "port" key if present, and the "dohpath" value.  (The DNS service's original port number MUST NOT be used.)

Clients SHOULD NOT query for any "HTTPS" RRs when using the constructed URI Template.  Instead, the SvcParams and address records associated with this SVCB record SHOULD be used for the HTTPS connection, with the same semantics as an HTTPS RR.  However, for consistency, service operators SHOULD publish an equivalent HTTPS RR, especially if clients might learn this URI Template through a different channel.

# Limitations

This document is concerned exclusively with the DNS transport, and does not affect or inform the construction or interpretation of DNS messages.  For example, nothing in this document indicates whether the service is intended for use as a recursive or authoritative DNS server.  Clients must know the intended use in their context.

# Relationship to DNS URIs

The `dns:` URI scheme {{?DNSURI=RFC4501}} describes a way to represent DNS queries as URIs.  This scheme optionally includes an authority, comprised of a host and port number (with a default of 53).  DNS URIs normally omit the authority, or specify an IP address, but a hostname is allowed, in which case it is suitable for use with this mapping.

# Examples

* A resolver at `simple.example` that supports DNS over TLS on port 853 (implicitly, as this is the default protocol and its default port):

      _dns.simple.example. 7200 IN SVCB 1 simple.example.

* A resolver at `doh.example` that supports only DNS over HTTPS (DNS over TLS is disabled):
      _dns.doh.example. 7200 IN SVCB 1 doh.example. ( no-default-alpn
        alpn=h2 dohpath=/dns-query{?dns} )

* A resolver at `resolver.example` that supports
  * DNS over TLS on `resolver.example` ports 853 (implicit in record 1) and 8530 (explicit in record 2), with `resolver.example` as the Authentication Domain Name,
  * DNS over HTTPS at `https://resolver.example/dns-query{?dns}` (record 1), and
  * an experimental protocol on `fooexp.resolver.example:5353` (record 3):

        $ORIGIN resolver.example.
        _dns 7200 IN SVCB 1 @ alpn=h2,h3 dohpath=/dns-query{?dns}
                     SVCB 2 @ port=8530
                     SVCB 3 fooexp ( port=5353
          alpn=foo no-default-alpn foo-info=... )

* A nameserver at `ns.example` whose service configuration is published on a different domain:

      $ORIGIN example.
      _dns.ns 7200 IN SVCB 0 _dns.ns.nic

# Security Considerations

## Adversary on the query path

This section considers an adversary who can add or remove responses to the SVCB query.

Clients MUST authenticate the server to its name during secure transport establishment.  This name is the hostname used to construct the original SVCB query, and cannot be influenced by the SVCB record contents.  Accordingly, this draft does not mandate the use of DNSSEC.  This draft also does not specify how clients authenticate the name (e.g. selection of roots of trust), which might vary according to the context.

Although this adversary cannot alter the authentication name of the service, it does have control of the port number and "dohpath" value.  As a result, the adversary can direct DNS queries for $HOSTNAME to any port on $HOSTNAME, and any path on "https://$HOSTNAME", even if $HOSTNAME is not actually a DNS server.  If the DNS client uses shared TLS or HTTP state, the client could be correctly authenticated (e.g. using a TLS client certificate or HTTP cookie).

This behavior creates a number of possible attacks for certain server configurations.  For example, if "https://$HOSTNAME/upload" accepts any POST request as a public file upload, the adversary could forge a SVCB record containing `dohpath=/upload`.  This would cause the client to upload and publish every query, resulting in unexpected storage costs for the server and privacy loss for the client.

To mitigate this attack, a client of this SVCB mapping MUST NOT provide client authentication for DNS queries, except to servers that it specifically knows are not vulnerable to such attacks, and a DoH service operator MUST ensure that all unauthenticated DoH requests to its origin maintain the DoH service's privacy guarantees, regardless of the path.  Also, if an alternative service endpoint sends an invalid response to a DNS query, the client SHOULD NOT send more queries to that endpoint.

## Adversary on the transport path

This section considers an adversary who can modify network traffic between the client and the SvcDomainName (i.e. the destination server).

For a SVCB-reliant client ({{SVCB}} Section 3), this adversary can only cause a denial of service.  However, because DNS is unencrypted by default, this adversary can execute a downgrade attack against SVCB-optional clients.  Accordingly, when use of this specification is optional, clients SHOULD switch to SVCB-reliant behavior if SVCB resolution succeeds.  Specifications making using of this mapping MAY adjust this fallback behavior to suit their requirements.

# IANA Considerations

Per {{SVCB}} IANA would be directed to add the following entry to the SVCB Service Parameters registry.

| Number  | Name    | Meaning                      | Reference       |
| ------- | ------- | ---------------------------- | --------------- |
| TBD     | dohpath | DNS over HTTPS path template | (This document) |

Per {{?Attrleaf=RFC8552}}, IANA would be directed to add the following entry to the DNS Underscore Global Scoped Entry Registry:

| RR TYPE | _NODE NAME | Meaning       | Reference       |
| ------- | ---------- | ------------- | --------------- |
| SVCB    | _dns       | DNS SVCB info | (This document) |


--- back

# Mapping Summary

This table serves as a non-normative summary of the DNS mapping for SVCB.

|                                  |                                        |
| -------------------------------- | -------------------------------------- |
| **Mapped scheme**                | "dns"                                  |
| **RR type**                      | SVCB (64)                              |
| **Name prefix**                  | `_dns` for port 53, else `_$PORT._dns` |
| **Automatically Mandatory Keys** | `port`, `no-default-alpn`              |
| **SvcParam defaults**            | `alpn`: \["dot"\]                      |
| **Special behaviors**            | Supports all HTTPS RR SvcParamKeys     |
|                                  | Overrides the HTTPS RR for DoH         |
|                                  | Default port is per-transport          |
|                                  | No encrypted -> cleartext fallback     |

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
