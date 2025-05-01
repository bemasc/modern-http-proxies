---
title: "Template-Driven HTTP Request Proxying"
abbrev: "Templated HTTP Request Proxies"
category: std

docname: draft-schwartz-modern-http-proxies-latest
ipr: trust200902
area: art
submissiontype: IETF
workgroup: httpbis
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net

normative:

informative:


--- abstract

HTTP request proxying behaviors have long been part of the core HTTP specification.  However, the core request proxying functionality has several important deficiencies in modern HTTP environments.  This specification defines an alternative proxy service configuration for HTTP requests.  The proxy service is identified by a URI Template, similarly to "connect-tcp" and "connect-udp".

--- middle

# Introduction

## History

An HTTP forward proxy (or just "proxy" in the HTTP standards) is an HTTP service that acts on behalf of the client as an intermediary for some or all HTTP requests.  HTTP/1.0 defined the initial HTTP proxying mechanism: the client formats its request target in "absolute form" (i.e., with a full URI in the Request-Line) and delivers it to the proxy, which reissues the request to the origin specified in the URI ({{?RFC1945, Section 5.1.2}}).  In this specification, we call this behavior a "classic HTTP request proxy".

In HTTP/1.1, proxy requests are additionally required to carry a Host header whose value matches the authority in the Request URI (not the name of the proxy server).  In HTTP/2 and HTTP/3, the destination host is specified in the :authority pseudo-header field ({{?RFC9113, Section 8.3.1}}).

## Problems

HTTP clients can be configured to use proxies by selecting a proxy host, a port, and whether to use a security protocol. However, requests to the proxy do not carry this configuration information. Instead, they only indicate the URI of the requested resource. This prevents any HTTP server from hosting multiple distinct proxy services, as the server cannot distinguish them by path (as with distinct resources) or by origin (as in "virtual hosting").

The absence of an explicit origin for the proxy also rules out the usual defenses against server port misdirection attacks (see {{Section 7.4 of ?RFC9110}}).

<!--
### Context Mixing

Classic HTTP request proxies forward the entire request, including all its headers, with a few exceptions:

* The Proxy-Authenticate, Proxy-Authorization, Proxy-Authentication-Info, and Proxy-Status headers are not forwarded.
* In HTTP/1.1, the Host header, and connection-specific headers listed in Connection header, are not forwarded.
* Proxies are required to add a Via header identifying the proxy.
* For TRACE and OPTIONS methods, proxies are required to check and decrement the Max-Forwards header.

In HTTP/2 and HTTP/3, this leaves no way to attach metadata to the request between the client and the proxy.
-->

## Overview

This specification describes an alternative protocol for an HTTP request proxy.  Like CONNECT-TCP, CONNECT-UDP, and CONNECT-IP, the proxy is identified by a URI Template.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Requirements

## Use of Templates

A templated HTTP request proxy is identified by a URI Template containing a variable named "target_uri".  To convert an HTTP request into a proxied request, the client MUST substitute the request's URI into this variable, expand the template, and use the result as the new request URI.

HTTP headers and status codes are processed in the same way as in classic HTTP request proxies.

A templated HTTP request proxy is also suitable for use as an Oblivious HTTP relay, if it provides the required privacy guarantees.

## Configuration

Clients that support both classic HTTP request proxies and template-driven proxies MAY accept both types via a single configuration string.  If the configuration string can be parsed as a URI Template containing the "target_uri" variable, it is a template-driven request proxy.  Otherwise, it is presumed to represent a classic HTTP request proxy.

This specification defines a new Proxy-Status parameter: "use_template" (see registration in {{iana-considerations}}), which conveys a preferred URI Template for the proxy.  Upon receipt of this parameter, the client SHOULD update its configuration to use the new template for the remainder of the session if possible, and retry the request using the new template if it also received a Proxy-Status "error" parameter.

If the client is configured with a classic HTTP request proxy, and the template string is the special value "default", the client MUST use the following default proxy template:

~~~
https://$PROXY_HOST:$PROXY_PORT/.well-known/masque
                 /http/{target_uri}
~~~
{: title="Registered default template"}

This allows a virtual-hosted proxy server to learn the proxy's hostname, which is not present in the initial request.

# Examples

Consider a proxy identified as "https://example.com/proxy{?target_uri}".  Requests would then be transformed as follows:

~~~
Original request:
PATCH /resource HTTP/1.1
Host: api.example
Content-Type: application/example
...

Transformed request:
PATCH /proxy?target_uri=https%3A%2F%2Fapi.example%2Fresource HTTP/1.1
Host: example.com
Content-Type: application/example
Proxy-Authorization: ...
...
~~~

Notes on this example:

* The HTTP method is not altered.
* The request-related headers such as Content-Type are preserved, but the Host header (or :authority in HTTP/2 and HTTP/3) is altered.
* Certain characters in the target URI are percent-encoded during URI Template expansion.
* The scheme, which is implicit in the original request, is explicit in the transformed request.  The scheme in this example is "https", indicating that the client is asking the proxy to establish a secure connection to the target.
* The client can add Proxy-* headers to communicate with the proxy.

A templated HTTP request proxy can be used as an Oblivious HTTP Relay.  For example, suppose the relay is identified as "https://proxy.example.org/relay{?target_uri}", and the Oblivious HTTP Gateway is "https://example.com/gateway".  The client would send requests to the proxy as follows:

~~~ http-message
POST /relay?target_uri=https%3A%2F%2Fexample.com%2Fgateway HTTP/1.1
Host: proxy.example.org
Proxy-Authorization: ...
Content-Type: message/ohttp-req
...
~~~
{: title="Use of an HTTP request proxy as an Oblivious relay"}

If a templated HTTP request proxy supports HTTP/2 and Extended CONNECT, it is even possible to reach a CONNECT-TCP transport proxy through it:

~~~ http-message
CONNECT HTTP/2.0
:authority = request-proxy.example
:scheme = https
:path = /proxy?target_uri=https%3A%2F%2Ftransport-proxy.example%2Fproxy
    %3Ftarget_host%3Ddestination.example%26target_port%3D443
:protocol = connect-tcp
capsule-protocol: ?1
...
~~~
{: title="Use of a TCP transport proxy through an HTTP request proxy"}

A proxy can use the "use_template" proxy status error to reconfigure existing clients:

~~~ http-message
GET /main?target_uri=https%3A%2F%2Fexample.com%2F HTTP/1.1
Host: proxy.example.org
Proxy-Authorization: ...
Content-Type: message/ohttp-req
...

HTTP/1.1 200 OK
Proxy-Status: proxy.example.org; \
    use_template="https://proxy.example.org/beta{?target_uri}"; \
    details="You have been assigned to the beta test group"
Content-Type: message/ohttp-resp
...
~~~
{: title="Updating the configured template with 'use_template'"}

If the client's proxy configuration string was "proxy-foo.example.org:54321", the client will start by issuing a classic HTTP proxy request, but the proxy can use the "use_template" parameter to inform the client that it should use templated requests instead:

~~~ http-message
GET https://example.com/ HTTP/1.1
Proxy-Authorization: ...
Host: example.com
Accept: text/html

HTTP/1.1 400 Bad Request
Proxy-Status: proxy-no-template.example.org; \
    use_template="default"; \
    error="http_request_denied"; \
    details="Proxy template required"

GET /http/https%3A%2F%2Fexample.com%2F HTTP/1.1
Proxy-Authorization: ...
Host: proxy-foo.example.org:54321
Accept: text/html

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: ...
...
~~~
{: title="Using 'use_template' to upgrade from classic to templated proxying"}

# Security Considerations

None

# Operational Considerations

Templated HTTP proxies can make use of standard HTTP gateways and path-routing to ease implementation and allow use of shared infrastructure.  To be compatible, a gateway must forward Proxy-* request headers to the origin.

# IANA Considerations

IF APPROVED, IANA is requested to add the following entry to the "MASQUE URI Suffixes" registry:

| Path Segment | Description           | Reference       |
| ------------ | --------------------- | --------------- |
| http         | HTTP Request Proxying | (This document) |

IF APPROVED, IANA is requested to add the following entry to the "HTTP Proxy Status Parameters" registry:

* Name: use_template
* Description: A URI Template that should be used for requests to this proxy server.  The special value "default" indicates that the client should use the default template for the configured proxy hostname (which is not known to the proxy).
* Reference: (This document)

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
