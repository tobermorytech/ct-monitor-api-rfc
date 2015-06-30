---
coding: UTF-8

title: A RESTful API for Certificate Transparency Monitors
docname: draft-palmer-ct-monitor-api-latest
date: 2015-06-15

author:
  -
    ins: M. Palmer
    name: Matt Palmer
    org: Tobermory Technology
    email: matt@tobermorytech.com
  -
    ins: R. Stradling
    name: Rob Stradling
    org: Comodo CA, Ltd.
    email: rob.stradling@comodo.com

normative:
  RFC2119:

informative:
  RFC6962:
  I-D.ietf-trans-rfc6962-bis:
  REST:
    target: http://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf
    title: Architectural Styles and the Design of Network-based Software Architectures
    author:
      ins: R. Fielding
      name: Roy Fielding
      org: University of California, Irvine
    date: 2000
    format:
      PDF: http://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf
  I-D.draft-linus-trans-gossip-ct-01: gossip
  I-D.draft-ietf-trans-threat-analysis-00: threats
  WHAT-IS-CT:
    target: http://www.certificate-transparency.org/what-is-ct
    title: What is Certificate Transparency?

stand_alone: true
pi: [toc, sortrefs, symrefs]

--- abstract

An important component of the Certificate Transparency ecosystem is that of
the "monitor", a service which examines the entries made in Certificate
Transparency logs and reports on anomalies.  They provide an important link
between the bulk data contained within logs, and end users who need to know
about what the logs contain.

In order to benefit fully from the functionality provided by logs, it is
useful that monitors can be accessed programmatically.  Further, it is a
benefit to end users if they can interact with multiple independent monitors
using the same client code, without having to write customised programs for
each monitor they wish to interact with.

The purpose of this specification is to provide standard semantics of a
variety of operations which can be performed against a monitor, using a
RESTful HTTP(S) API.

--- middle


# Terminology

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 {{RFC2119}}.


## Glossary

Certificate Transparency
: A scheme for providing for post-hoc examination of the issuance of X.509
  certificates in the Internet PKI, as defined by {{RFC6962}} and
  {{I-D.ietf-trans-rfc6962-bis}}.  It may be extended in the future to provide
  transparency to other trust-related Internet technologies, such as DNSSEC.

Client
: Any person or program which is authorised to make use of the services
  provided by a Certificate Transparency Monitor.

Maximum Merge Delay (MMD)
: A time period, declared by a log, within which any certificate submitted
  to and accepted by the log is guaranteed to have been added to the log and
  covered by a Signed Tree Head (STH).

Monitor
: A service provided to one or more clients which verifies the behaviour of
  Certificate Transparency logs, and watches the logs for certificates of
  interest to those clients.  See also {{RFC6962}} section 5.3, and
  {{WHAT-IS-CT}}.

Signed Tree Head (STH)
: A data structure containing a timestamp, a "tree head" hash (formed using
  the algorithm defined in {{RFC6962}} section 2.1), and a signature made by
  the log's private key.


# Desired Features

Certificate Transparency Monitors contain information of interest to several
distinct classes of user.  The API should provide a straightforward means of
satisfying these tasks, whilst still providing flexibility to allow novel
uses of the interface.

To help in identifying use cases, we have identified a number of
"stakeholder archetypes", which help to ensure we have fully enumerated a
complete set of use cases.


## Monitor Operator

The operator of a monitor system is a rather important stakeholder, insofar
as they are responsible for providing the service, maintaining its
availability and integrity, and paying the bills.


### Authenticating Users

Some monitors may be operated partially or completely on a "subscription only"
basis -- that is, the monitor is not entirely available for use by the
general public anonymously.

In this circumstance, it is important that the user of the monitor service
be able to authenticate themselves, so that the monitor service knows who it
is interacting with.


### Publishing Available Monitor Features

Some monitors may wish to only provide a subset of all specified aspects of
the CT monitor interface.  Alternately, they may wish to provide additional
features which are not yet available in all implementations.  To allow
clients to discover which features a particular monitor supports, there must
be some way of publishing that list.


### Preventing Abuse


#### Rate-limiting Requests

A monitor will almost certainly want to be able to limit the number of requests
that a single client can make in a given time period.


#### Terminating Long-running Requests

Some requests may be difficult or even impossible for a monitor to handle, due
to the volume of data that must be processed.  Since it may not be possible for
a monitor to detect these requests before attempting to process them, a monitor
will almost certainly want to be able to watch for and terminate long-running
requests.


## Log Stalker

This client is interested in looking for logs that are failing to meet their
obligations under ordinary CT policies.  That includes failing to uphold
their MMD (Maximum Merge Delay), providing bad signatures on timestamps or
tree heads, and other behaviour which is indicative of bad behaviour on the
part of a *log* (as opposed to a CA or other PKI party).


### See Which Logs Are Monitored

There's no point in using a monitor to stalk a log if that log isn't being
monitored.


### Verifying Correctness of The Monitor's Signed Tree Heads

The log stalker is interested in the current and past STHs which were
provided by the log, and collected by the monitor.  Specifically, whether or
not (a) the signature validated, and (b) the tree head hash was correct
based on the contents of the log entries which the tree head covers.  It is
also important that the time at which the monitor retrieved the tree head is
available, in order for the stalker to be able to verify that the MMD of the
log has not been violated.


### Verifying Correctness of an Externally Obtained Signed Tree Head

If the log stalker has collected STHs of zer own, they may wish to ask the
monitor to verify the head hash against the log entries which the monitor
has retrieved.


## Site Operator

A person responsible for a website (or other TLS-using service) wants to
discover misissuance relating to domain names they are responsible for, and
possibly other misissuance involving their organisation.


### Find All Certs For a Domain

To find historical misissuance, a site operator will want to ask the monitor
to retrieve certificates which would be considered valid for a specific domain
(including wildcards), or for a subtree of the DNS namespace.

It is possible that a particularly well-organised site operator may have a
list of all legitimately issued certificates, and may wish to exclude
certificates from being returned, based on criteria such as issuer/serial
tuples.

As time passes, the CT logs will contain an ever increasing number of expired
certificates.  A site operator will almost certainly want to have the option of
excluding expired certificates from being returned, since misissued expired
certificates are less of a problem than misissued unexpired certificates.


### Be Alerted to New Issuance For a Domain

The canonical job for a log monitor is to alert a site operator to the
issuance of a new certificate which would be considered valid for a domain
or subtree of the DNS namespace.


### Manage Alert Registrations

A site operator may wish to modify the mechanism (e.g. SMS, email, RSS feed,
etc) by which they receive alert notifications, or to cancel alerts for a domain
or subtree that they are no longer interested in.

A site operator may want to register alerts for a large number of domains or
subtrees.  To keep track of these alert registrations, the site operator may
want to ask the monitor to return the complete list of alerts that they've
registered.

A site operator may also want alert notifications for multiple alert
registrations to be bundled together into at most one notification message per
configurable time period.

All operations to manage alert registrations should be authenticated, to avoid
the possibility of alert spamming (e.g. person A configures the monitor to send
unwanted alert notification emails to person B).


### Finding Certificates for an Organisation

This is similar to a per-domain search, however it acts on the other data
present in the certificate, such as the information in the Subject field.
Historical searches and real-time notifications will both need to be
available.


## Certification Authority

CAs are interested in independent visibility into what certificates are
being logged, to ensure that their CA certificate is not being misused.


### Finding Certificates Issued From a Given CA Certificate

This is a variation of the site operator searches, however instead of
providing a domain or keyword(s) to match, a CA certificate (or reference
thereto) is provided, and all certificates which were issued by that CA
certificate are returned.  This can either be a historical search or a
real-time alert.


## Trust Store Operator

The operators of a trust store, such as browser vendors, wish to have good
visibility into the entire "tree" of certificates which extend from a
specific root certificate, so as to make decisions about the way in which
the root CA has been operated.


### Finding Certificates Issued From a Given CA Certificate

This operation is practically identical to the CA version, however it may be
useful to be able to add a "depth" parameter to the search, to allow all
levels of a certificate "tree" to be retrieved simultaneously.


# RESTful Principles

The concepts of a "REST" architecture originally come from Roy Fielding's
Ph.D thesis {{REST}}, and have seen a great deal of varied interpretation
ever since.  To clarify what we mean by "RESTful" in the context of this
specification, it is worth enumerating the design principles used.


## Documents, not Data

All information exchanged between the monitor and client is formatted as
*documents* -- standalone representations of a cohesive set of facts.  The
semantics of each document type is defined by a media type definition.


## Relationships, not URLs

Rather than describe how to construct URLs to achieve certain desired
results, a REST architecture is built on links between documents.  This
implies there is an "entrypoint" URL, which serves a resource which provides
links to other resources.  The meaning of each link is defined by link
metadata known as the "relationship".
