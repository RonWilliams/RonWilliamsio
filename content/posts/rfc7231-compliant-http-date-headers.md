---
title: "RFC7231 Compliant HTTP Date Headers"
date: 2016-01-03T00:21:44-04:00
draft: false
type: "post"
---

> Tracking down the Date Header standard. Improve reverse-proxy and edge caching with proper HTTP Date header usage.

## PROJECT BACKGROUND

Over the past five months, I have worked on the USDA WCMaaS (Web Content Management as a Service) Platform helping migrate and launch several Drupal and static based websites on the environment. WCMaaS is a Salt managed and Docker containerized system intended for use across USDA for improving sites. Inline with other modern tools selected, the system uses Varnish 4.x (version is important later!). We've gone from site launches requiring months to our recent launch being 16 days from dev delivery to production. Overall it has been rewarding and we're now implementing changes on an agile basis including Solr and reliable Akamai profiles. Within the Akamai Luna control panel I noticed that the hit-offload metric was very low (5%) which meant less than an ideal amount of traffic was being handle by the edge caching provided by Akamai's service.

## INVESTIGATING HTTP RESPONSE HEADERS

After opening a ticket with Akamai, they tracked down some of the issue to the HTTP Date response header returned by the origin servers. When returning the Date header using curl (some headers redacted for clarity), the field content returned a range of 1-15 minutes in the past.

```
$ curl -sI https://example.gov/sites/default/files/file.pdf -H "Host: farmtoschoolcensus.fns.usda.gov"
HTTP/1.1 200 OK
Date: Sun, 20 Dec 2015 13:00:41 GMT
Last-Modified: Mon, 23 Nov 2015 15:12:39 GMT
Cache-Control: max-age=1209600
Expires: Sun, 03 Jan 2016 13:00:41 GMT
X-Varnish: 8614352 6450650
Age: 196430
Via: 1.1 varnish-v4
X-Cache: HIT
```

## IS SOMETHING WRONG WITH VARNISH?

With the information provided by Akamai, I set out to find out why Varnish was setting an old date header. For my personal server I have Varnish 3.x and the same commands against that system return a more immediate timestamp compared to Varnish 4.x. In 2014, Varnish devs intentionally changed the Date header behavior to [use the Date header provided by the backend](https://www.varnish-cache.org/trac/changeset/89870e0bbd785964c322e1e453f492d747731c88/) (in my case, Docker containers running Apache). A [Varnish Date header issue](https://www.varnish-cache.org/trac/ticket/1673) was closed and marked Won't Fix and referred to the original commit. Within the commit was a reference to [RFC2616](https://www.ietf.org/rfc/rfc2616.txt), the old circa 1999 web standard document that was incredibly formative to the internet, but could not predict all potential implementations or new technologies.

## RESEARCHING THE RFC2616/7231 STANDARDS

With a lead from the Varnish issue queue, I reviewed RFC 2616 then found section 14.18:

```
   The Date general-header field represents the date and time at which
   the message was originated, having the same semantics as orig-date in
   RFC 822. The field value is an HTTP-date, as described in section
   3.3.1; it MUST be sent in RFC 1123 [8]-date format.
 
       Date  = "Date" ":" HTTP-date
 
   An example is
 
       Date: Tue, 15 Nov 1994 08:12:31 GMT
 
   Origin servers MUST include a Date header field in all responses,
   except in these cases:
```

I’ve drawn attention to orig-date which is defined in [RFC822](https://www.ietf.org/rfc/rfc0822.txt) (a 1982 document!) as ‘original’. I looked for a newer spec since the RFC822 standards doc was a bit sparse and the [IETF HTTP Working Group chair recommends against RFC2616](https://www.mnot.net/blog/2014/06/07/rfc2616_is_dead). [RFC7231](https://httpwg.github.io/specs/rfc7231.html#header.date) provides a similar definition, but instead refers to [RFC5322](https://tools.ietf.org/html/rfc5322#section-3.6.1) as a definition of the orig-date:

```
   The origination date specifies the date and time at which the creator
   of the message indicated that the message was complete and ready to
   enter the mail delivery system.  For instance, this might be the time
   that a user pushes the "send" or "submit" button in an application
   program.  In any case, it is specifically not intended to convey the
   time that the message is actually transported, but rather the time at
   which the human or other creator of the message has put the message
   into its final form, ready for transport.  (For example, a portable
   computer user who is not connected to a network might queue a message
   for delivery.  The origination date is intended to contain the date
   and time that the user queued the message, not the time when the user
   connected to the network to send the message.)
```

## FINAL VERDICT: CACHES SHOULD USE BACKEND DATE HEADER CONTENT

While standards are subject to interpretation, the backend/origin Date header should be used by caches (edge and reverse proxy). I do believe that the Varnish mechanism of passing on the backend request headers is appropriate. I'll be commenting on the Varnish issue so future users won't research as much to find out what the Varnish devs have already concluded. In addition we're working with Akamai to update our cache profiles to use the origin Date header content as defined in the specs.
