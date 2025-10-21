---
layout: post
title:  "POST Mortem: How Azure Application Gateways's Missing 308 Killed Our Linked Data API"
date:   "2025-10-21"
categories: [programming, tech, w3c, linked-data, restful, http, wqa-api, microsoft, azure, application-gateway]
---

In the Linked Data world, cool urls don't change. That means in the RDF world you're coining URIs that should be resolvable, and you pick the easiest one. Most people in the linked data world use `http://` when coining URIs, even though today's modern internet lives on `https://` with upgrades handled by the service's web stack.

The Water Quality service launching on the environment.data.gov.uk portal has a RESTful, Hydra API, and it supports a combination of `GET` and `POST` methods to retrieve data. The most useful endpoints living at `/data` are `POST`, as they can receive GeoJSON bounding boxes to query both geographic and observation data, though some uses don't require a body.

In our testing we discovered that Python clients break when navigating the pagination of our service, but JavaScript works. WTF?

## The HTTP Redirect Status Code Landscape

The 300 series of HTTP Status Codes defined in RFC 7231, with 308 added in RFC 7538 help people navigate the internet automatically when resources move, protocols change, and they're all quite useful.

- **301 (Moved Permanently)**: The old guard—allows method changes
- **302 (Found)**: Temporary and method-flexible
- **303 (See Other)**: Forces GET (useful for POST-Redirect-GET pattern)
- **307 (Temporary Redirect)**: Preserves method but temporary semantics
- **308 (Permanent Redirect)**: The hero we need—permanent + method preservation

The issue It's not a resource **move** (different URI), it's a protocol **upgrade** (same resource, different scheme).

## Link Data APIs need 308

Our canonical URIs often use `http://` scheme as protocol agnostic identifiers; however transport security requires HTTPS. Content negotiation and RDF payloads reference http:// URIs, and we don't select the protocol on the fly in our responses. Both the link headers and the Hydra pagination links in our endpoint use the same URIs to help people navigate our pagination setup.

So you see how this is going to go? With `POST` getting redirected to `GET` will cause things to fall over when we erroneously get a 301 from Microsoft's Application Gateway?

## The Azure Application Gateway Gap

The current available responses for a HTTP to HTTPS upgrade in Azure's Application Gateway service are 301, 302, 304, and 307. It's missing the semantically accurate and method-preserving 308. Not only that, we can't target specific paths or entry points in the service. We are forced to chose between wrong semantics (i.e. temporary redirects) or broken clients (`POST` gets converted to `GET`).

## Real-World Impact: Client Behaviour Broken

Let's be honest, the problem here is that Python is full of pedants (see: Pydantic), and the interpretation of RFC 7231 by the authors of its `requests` library have correctly implemented their redirect flag in the `post()` method. When a 301 redirect is encountered, `requests` converts the POST to a GET—which our `/data` endpoint doesn't support, returning a **405 Method Not Allowed** error.

What should be a simple for loop navigating the link headers to collect a paginated dataset now requires custom redirect handling. What should be the simple contents of the `while next_url:` loop.

```python
# What breaks with 301:
response = requests.post(next_url, headers=headers, data="")
# requests converts POST → GET on 301 redirect
# Server responds: 405 Method Not Allowed
# Pagination fails immediately
```

Becomes the more convoluted:

```python
# Manual redirect handling to preserve POST method:
response = requests.post(
    next_url, 
    headers=headers, 
    auth=auth, 
    data="", 
    allow_redirects=False  # Disable automatic redirect
)

# Handle redirect manually to keep POST
if response.status_code in (301, 302, 307, 308):
    next_url = response.headers['Location']
    continue  # Re-POST to new URL
```

Now I have to build my own redirect handling in Python because Microsoft has the semantics of their response codes wrong. I'm fine with it, but I want people to be able to be able to use our endpoint easily.

Our front-end developers didn't experience the same problem, which means that JavaScript's fetch doesn't do the same thing. This gives us an inconsistent API experience, and even with documentation being clear what's going wrong with their code I'm still going to get support tickets that the thing is broken.

## Microsoft: Fix Your Shit

Your Application Gateway redirect options aren't complete. Give us a `308` code, allow us to be the pedants I want us to be. It would make a massive impact for the semantic web, improve our RESTful APIs, and follow modern HTTP patterns without breaking it for everyone else.

Standards exist for a reason; it's not a niche concern, as LLMs and Agentic AI usage becomes more and more common, having modern ways of accessing knowledge graphs and FAIR Data requires getting the semantics right everywhere — including in our HTTP response codes.

@Azure: gimme the response code 308.

---

*__Note__:* I have a support request asking for this behaviour. I expect Microsoft to change nothing.