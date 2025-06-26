---
layout: post
title:  "PostgreSQL is the best triplestore"
date:   "2025-06-26"
categories: [programming, tech, w3c, linked-data, rdf, json-ld]
---

When folks want data they went it on a subject basis, making providing linked data that much easier. I have been exploring using Postgres with FastAPI and pydantic to serialize JSON-LD direct from SQL, to give users a familiar JSON RESTful API with content negotiated RDF baked in.

Compared to the existing Jena-based API its throughput is two orders of magnitude faster, more reliable, and the data ingress doesn't make me reconsider being in this domain.

I hinted about this in February with a post about [JSON-LD and prefixes](./2025-02-20-jsonld-and-prefixes.html). The API should be finished by the end of the month.

(I'm on my second of hopefully two refactors.)