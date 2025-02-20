---
layout: post
title:  "A small gotchya around jsonld 1.0, 1.1, and gen-delims from RFC3986"
date:   "2025-02-20"
categories: [programming, tech, w3c, linked-data, rdf, json-ld]
---

I'm working on an exciting project which is the implementation of the [SOSA standard](https://w3c.github.io/sdw-sosa-ssn/ssn/), and as part of the project I wanted to use the [envo ontology](http://environmentontology.org/) to provide context to the data contained therein.

As part of the development process I manually write RDF/Turtle to be sure I have the relationships correct for a single vertical component, and then I write a corresponding response in JSON-LD. From there I check for isomorphism of the two graphs to ensure that the JSON-LD is correct. The main reason why I manually serialize the JSON-LD is that the automatic conversion to JSON-LD is hideous; a well designed JSON-LD response could be indistinguishable from a resful JSON API response, but with the additional inclusions of various fields like `@context` and `@id` which can be ignored by users who would rather parse the data as a JSON object.

Now that I've set up the scene, let's get to the gotchya. I was writing a JSON-LD response for a vertical component, and I wanted to include the `envo` ontology as part of the context. I had the following JSON-LD:

```json
{
  "@context": {
    "envo": "http://purl.obolibrary.org/obo/ENVO_",
    "skos": "http://www.w3.org/2004/02/skos/core#"
  },
  "@id": "http://example.com/concept1",
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:closeMatch": "envo:00000022"
}
```

Which unfortunately serilaized to

```n3
<http://example.com/concept1> <http://www.w3.org/2004/02/skos/core#closeMatch> "envo:00000022" .
<http://example.com/concept1> <http://www.w3.org/2004/02/skos/core#prefLabel> "RIVER / RUNNING SURFACE WATER" .
```

As you can see we don't have the IRI of the `envo` object we expected (which would be `http://purl.obolibrary.org/obo/ENVO_00000022`).

Turns out that JSON-LD 1.0 doesn't support ending what in TTL is called [`@prefix`](https://www.w3.org/TR/turtle/#prefixed-name) when it doesn't end in a gen-delim character as defined in [RFC3986](https://datatracker.ietf.org/doc/html/rfc3986). tl;dr `_`s are out in prefixes with JSON-LD 1.0; however...

JSON-LD 1.1 adopts the more permissive approach of IRI creation like RDF/TTL with a bit more in the context. By putting the prefix in an object's `@id`, and setting a `@prefix` keyword to `true`, I finally got the result I wanted.

```json
{
  "@context": {
    "@version": 1.1,
    "envo": {
      "@id": "http://purl.obolibrary.org/obo/ENVO_",
      "@prefix": true
    },
    "skos": "http://www.w3.org/2004/02/skos/core#"
  },
  "@id": "http://example.com/concept1",
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:closeMatch": "envo:00000022"
}
```

I may be breaking some backwards compatibility here; for example neither the [JSON-LD Playground](https://json-ld.org/playground/) nor the [EasyRDF Converter](https://www.easyrdf.org/converter) serialize it differently to the first example in n3; however the likes of rdflib 7.1.3 does.

For me the JSON-LD "shape" is more important than giving supporting JSON-LD 1.0; and I can't change the `envo` prefix. Necessity breeds breaking backwards compatibility. 