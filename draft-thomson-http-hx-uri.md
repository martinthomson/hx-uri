---
title: Identifying HTTP Messages with URIs
abbrev: HTTP Message URIs
docname: draft-thomson-http-hx-uri-latest
category: exp

ipr: trust200902
area: ART
workgroup: HTTPbis
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net


--- abstract

URI schemes are defined that enable identification of HTTP exchanges, or parts
of those exchanges.


--- middle

# Introduction

It is common for applications that use HTTP {{!HTTP=I-D.ietf-httpbis-semantics}}
to use a "follow your nose" design.  In this design, clients make requests to
discover or create resources and to learn information about resources they are
interested in.  Once the identity of resources is learned, clients then interact
with those resources.

A negative consequence of these designs is that the discovery or creation steps
add latency to any operation that depends on the identity of resources.

For applications that use well-defined formats, though the result of a request
might be unknown, an application might have reliable knowledge about the form of
the response.  If components of that answer could be incorporated into another
request by reference, then the application might save a round trip for every
such occurrence.

The `hx` URI scheme identifies components of HTTP exchanges.  The `hxr` URI
scheme provides for further indirection, dereferencing components of HTTP
exchanges that contain URIs and further dereferencing them.


## Example

In this hypothetical example, a client wishes to find and update a resource.

```
POST /find-object?name=example HTTP/1.1
Host: example.com

```

The response provides a location for the resource:

```
HTTP/1.1 200 OK
Content-Type: example/example+json

{
  "uri": "https://example.com/roZ2ITW",
  "name": "example",
  "items": { "a": 1, "b": 2 }
}
```

After receiving the identity of the resource, the client can then interact with
that resource, here copying the value of "b" to a new key called "c":

```
POST /roZ2ITW HTTP/1.1
Host: example.com

add_item: c=2
```

With an `hx` URI, and support from the server, the client can send the second
request at the same time as the first, relying on the server to dereference the
`hx` URI:

```
POST hxr:///0/a/b#/uri HTTP/1.1
Host: example.com

add_item: c=hx:///0/a/b#/items/b
```

If the server understands the `hxr` URI scheme, it dereferences that URI to
determine the target of the request.  The value from /items/b (using JSON
Pointer {{?RFC6901}}) is copied using the `hx` URI scheme.

Note that it can be seen that though the initial POST request that establishes
the transaction is not idempotent, the client is able to construct subsequent
requests in a way that ensures that they are conditional on the outcome of that
request.  This ensures that the transaction does not complete unless all
requests are successful.


## Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


## Terminology

This document uses terminology from HTTP {{!HTTP}}.  The phrase HTTP URI is used
to refer to http:// and https:// URIs collectively.  This document is only
capable of identifying requests that are sent over secured transports.


# Overview

An `hx` URI identifies an HTTP exchange, or parts of that exchange.  The primary
advantage of this in referring to the product of a request before it is
completed.

A scheme of "hx" is followed by an authority that identifies the connection on
which the exchange was initiated.  A minimal path includes an identifier for the
exchange, as a decimal number.  For instance, assuming that the authority
`b5dd5901aef3f33de572` refers to an HTTP/2 connection, the following URI
identifies entire exchange on stream 7 of that connection.

```
hx://b5dd5901aef3f33de572/7
```

Adding additional path elements narrows this to refer to the request:

```
hx://b5dd5901aef3f33de572/7/q
```

Or specific components of the response, such as a Location header field value:

```
hx://b5dd5901aef3f33de572/7/a/h/location
```

To ensure that the Location header field is only used if the request resulted in
the creation of a new resource (that is, the response had a 201 (Created) status
code), conditions can be added to the URI as query parameters:

```
hx://b5dd5901aef3f33de572/7/a/h/location?201
```

A fragment can be used if the content has an associated content type, which is
generally only possible for the body of a request or response:

```
hx://b5dd5901aef3f33de572/7/a/b#title
```

How a fragment is used depends on the content type of the identified resource,
so a condition might be added to specify the content type of the target
resource:

```
hx://b5dd5901aef3f33de572/7/a/b?ct=text%2Fhtml#title
```

The `hxr` URI scheme is identical to `hx` except that it is dereferenced twice.
An hxr reference can therefore be used where a URI would otherwise be used,
taking the value of the URI from the identified field in the identified
exchange.


# Authority

The authority component of an `hx` or `hxr` URI is an identifier for a
connection.

The process for generating a unique identifier uses TLS exporters (see Section
7.5 of {{!TLS13=RFC8446}}).  Consequently, exchanges on connections that do not
use TLS cannot be identified using `hx` or `hxr` URIs.

A TLS exporter with the label "EXPORTER-hx-authority" and an empty context is
used to produce a 10 octet value.  This value is then encoded in hexadecimal (or
Base 16 {{!RFC4648}}) to produce a 20 character authority.

The authority component of an can be omitted where the URI is exchanged over the
same connection.

```
hx:///7
```

The userinfo and port components of an `hx` or `hxr` URI MUST NOT be used.  Any
URI with userinfo or port components is invalid.


# Identifying an Exchange {#exchange}

After identifying the connection, the first element of the path after the
initial slash ("/") identifies a request-response exchange.

A decimal value is used to identify an exchange.  How requests are identified
depends on the version of HTTP in use.

A single "p" character followed by a decimal value is used to identify a server
push, see {{exchange-push}}.

An `hx` or `hxr` URI always includes fields that identify a request.  Thus, the
following URIs are incomplete and therefore invalid:

```
hx://
hx:///
hx://b5dd5901aef3f33de572/
```


## Identifying HTTP/1.1 Exchanges {#exchange11}

In HTTP/1.1 {{!HTTP11=I-D.ietf-httpbis-messaging}} and earlier, the numeric
identifier for an exchange counts the number of exchanges on the connection that
precede the target exchange.  The first exchange on a connection is therefore
identified as `hx:///0`.  Subsequent requests increment this value by 1.


## Identifying HTTP/2 Exchanges {#exchange2}

In HTTP/2 {{!HTTP2=RFC7540}}, the numeric identifier for an exchange corresponds
to a HTTP/2 stream identifier.  The first exchange on a connection is therefore
identified as `hx:///1`.  As a result, all exchanges that are not server pushes
use odd-numbered identifiers.


## Identifiers HTTP/3 Exchanges (#exchange3}

In HTTP/3 {{!HTTP3=I-D.ietf-quic-http}}, the numeric identifier for an exchange
corresponds to a QUIC stream identifier.  The first exchange on a connection is
therefore identified as `hx:///0`.  Consequently, all exchanges that are not
server pushes use identifiers that are whole multiples of 4.


## Identifying Server Pushes {#exchange-push}

A server push exchange is identified by a "p" prefix followed by a decimal
value.  For example:

```
hx:///p6
```

In HTTP/2, a stream identifier is sufficient to distinguish between requests and
server pushes.  Thus, identifying a server push is possible even if the "p"
prefix is omitted.  In HTTP/2, all server pushes use even-numbered identifiers.

Server pushes in HTTP/3 are given a push ID, an identifier that might be the
same as the stream ID used for requests.  The push ID is used to identify server
push in HTTP/3.  Thus, in HTTP/3, the "p" prefix is necessary to properly
identify a server push.

HTTP versions prior to HTTP/2 do not provide server push, so an `hx` URI that
attempts to identify a server push cannot be successfully resolved.


# Targets {#target}

Without further qualification, an `hx` URI identifies a message exchange, both
the request and the response as a whole.  Applications can narrow this to a
request ({{target-request}}) or response ({{target-response}}).

A `hxr` URI always identifies a request or response.

## Identifying a Request {#target-request}

A request is identified by adding a "/q" to a URI identifying an exchange.  For
example:

```
hx://546c9bce274b06cf859d/5/q
```

## Identifying a Response {#target-response}

## Conditionally Identifying Responses by Status {#target-cond}


## Following Redirections


# Identifying Request or Response Components

## Identifying the Message Body

## Identifying a Represented Resource

## Identifying Header Field Values {#component-header}

## 1xx Responses and Trailers {#component-1xx)


# Conditions

The query string of an `hx` URI carries a set of conditions.  Unless any
conditions evaluate to true, the resolution of the URI will fail.  This allows
for specification of URIs that are conditional on details of the HTTP exchange.

For example, the following URI will not produce a result if the request is not
successful, ensuring that the body of a response like 503 is not used:

```
hx://b5dd5901aef3f33de572/7/a/b?2xx
```

Conditions are separated by the ampersand ("&") character.  Each comprises a
label that identifies the type of the condition, and an optional value.  The
value is separated from the label by an equals sign ("=") character.

This document defines conditions for status code ({{condition-status}}), header
field values ({{condition-header}}), and response content-type
({{condition-ct}}).  Conditions that are not understood always evaluate to
false, causing resolution to fail.


## Condition Percent-Encoding

The URI grammar {{!URI=RFC3986}} prohibits the use of certain characters in the
query string.  This scheme uses percent-encoding to allow conditions to carry
values that are not permitted by the URI grammar.  Section 2.1 of {{!URI}}
defines percent-encoding. The `hx-pct-encoded` rule in {{grammar-hx}} defines the
characters that are valid; all other values MUST be percent-encoded.


## Status Condition {#condition-status}

Any condition that starts with a numeral from "1" to "5" is used to specify a
condition on the response status.

If the condition contains three digits, the condition evaluates to true if the
response contained a matching status code.

A condition that contains a numeral and two "x" characters evaluates to true if
the status code is from the identified class.  Thus, "2xx" matches any
successful status code.

A condition that specifies an informational status code (1xx) will be true if an
informational response of that type was present.  It does not result in limiting
the components that can be selected.  A URI that identifies a header field will
resolve the final value of the header field, taking into account values from
final responses and trailers as defined in {#component-header}.

This condition can be used to identify components of a request, conditional on
the status code of the results.

New condition definitions MUST NOT start with a numeral from "1" and "5".


## Header Field Value Condition {#condition-header}

The header field condition is identified


This condition applies to any header field from the identified object.  Thus, if
the URI does not specify whether a request or response, the condition evaluates
to true based on the presence or value of the header or trailer field in request
or response.  If the target of the URI is a request or response, then only
header and trailer fields in the request or response (respectively) apply.  If
the URI identies the header, then only header fields are used to match.


## Response Content Type Condition {#condition-ct}

The response content type condition matches if the content type of the response matches the

The separating slash ("/") MUST be quoted in the value of this condition.

Unlike the header field condition ({{condition-header}}), the response content
type condition can be used to identify components of a request, conditional on
the status code of the results.


# hx URI Grammar {#grammar-hx}

In ABNF {{!RFC5234}}, the `hx` URI scheme can be described as a narrow profile
of that defined in {{!URI=RFC3986}}.

```
hx-URI = "hx://" [ hx-authority ] "/" hx-exchange
         [ "/" hx-targets ] [ "?" hx-conditions ]
hx-authority = 20HEXDIG
hx-exchange = [ "p" ] 1*DIGIT

hx-targets = hx-request / hx-response
hx-request = "q" [ "/" hx-component ]
hx-response = "a" [ "/" hx-component ]

hx-component = hx-header / hx-body / hx-trailer
hx-header = "h" [ "/" hx-token ]
hx-body = "b"
hx-trailer = "t" [ "/" hx-token ]
hx-token = "-" / "." / "_" / DIGIT / ALPHA

hx-conditions = hx-condition *("&" hx-condition)
hx-condition = hx-status-cond /
               hx-header-cond /
               hx-ct-cond /
               hx-extension-cond
hx-status-cond = ("1" / "2" / "3" / "4" / "5") (2DIGIT / "xx")
hx-header-cond = "h=" hx-token [ "=" hx-pct-encoded ]
hx-extension-cond = hx-token [ "=" hx-pct-encoded ]
```

<!--
hx-token could use the token rule from HTTP, but that would be stupid.
There is no good reason to include backticks in a header field name.
-->


# hxr URI Grammar {#grammar-hxr}

The `hxr` URI scheme uses the same basic grammar as the `hx` URI scheme.
However, since this can only ever reference parts of an exchange that could
contain a URI, the grammar is more narrowly defined.

```
hxr-URI = "hxr://" [ hx-authority ] "/" hx-exchange
         "/" hxr-targets [ "?" hx-conditions ]

hxr-targets = hxr-request / hxr-response
hxr-request = "q/" hxr-component
hxr-response = "a/" hxr-component

hxr-component = hxr-header / hx-body / hxr-trailer
hxr-header = "h" "/" hx-token
hxr-trailer = "t" "/" hx-token
```


# Security Considerations

Resolution of details of unfulfilled requests could present a significant state
commitment on servers.  Servers that receive requests that depend on other
requests might have to block processing until the outcome of the referenced
requests is complete.  Alternatively, servers might need to hold information
about completed requests in anticipation of receiving references to that
request.

Servers can fail resolution of `hx` or `hxr` URIs if the state required would
present an undue burden on their operation.  Servers might limit the types of
information that can be retained and referenced to reduce this cost.

Applications that use these URI schemes MUST define what types of reference a
server is expected to be able to handle, or provide a means of negotiating what
can be relied on.


# IANA Considerations

This document registers the `hx` and `hxr` URI schemes.  In support of this,
registrations are made in the TLS exporters registry ({{iana-exporter}}) and
registries are established for managing the parameters of the URI schemes
({{iana-registries}}).


## hx URI scheme Registration {#iana-scheme}

The `hx` URI scheme is registered according to the procedures in
{{!BCP35=RFC7595}}.

Scheme name:
: hx

Status:
: Permanent/Provisional

Applications/protocols that use this scheme name:

: Applications use URIs with this scheme to identify HTTP exchanges, requests,
  responses, or components of those messages.

Contact:
: IETF Chair <chair@ietf.org>

Change controller:
: IESG <iesg@ietf.org>

Reference:
: This document.


## hxr URI scheme Registration {#iana-scheme}

The `hxr` URI scheme is registered according to the procedures in
{{!BCP35=RFC7595}}.

Scheme name:
: hxr

Status:
: Permanent/Provisional

Applications/protocols that use this scheme name:

: Applications use URIs with this scheme in place of URIs where the intended URI
  is found in a component of an HTTP exchange.

Contact:
: IETF Chair <chair@ietf.org>

Change controller:
: IESG <iesg@ietf.org>

Reference:
: This document.


## TLS Exporter Registration {#iana-exporter}


## hx and hxr URI Scheme Registries {#iana-registries}


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
