---
layout: post
title:  "The Art of Semantic Procrastination: Why I Use Blank Nodes for Concepts That Aren't Mine"
date:   "2025-08-30"
categories: [programming, tech, w3c, linked-data, rdf, json-ld, wqa-api]
---

In the linked data world, there is always a temptation to boil the ocean. When building out a new API or even just a new dataset, there are so many concepts (`SKOS:Concept` and otherwise) that are undefined and uncoined which provide human context and you feel the pressure to define it in your RDF - at the risk of taking on too much and straying outside of your authority. I've faced that in the past while building out a linked data service at Office for National Statistics, and having been burnt by the numerous kettles we had going to define everything semantically. I've been determined to not make that mistake again.

The new API I've been developing for DEFRA is a [Hydra](https://www.hydra-cg.com)/[SOSA](https://w3c.github.io/sdw-sosa-ssn/ssn/) vocabulary based RESTful, content negotiated API for observational water quality data in England. The architecture of the service is FastAPI+PostGIS with a Next.JS frontend: the API doesn't know anything about RDF; however it responds via JSON-LD by default with the JSON written in a way that people not familiar with RDF would appreciate.

The main payload of the API is sampling points (`sosa:FeatureOfInterest`) have samples & samplings (`sosa:Sample`, `sosa:Sampling`), which in turn have observations (`sosa:Observation`). Each of these levels have domain-specific types, classifications, and annotations which are necessary for the interpretation and discovery of these data; however no authoritative, public resource currently exists of these concepts.

![A felted shrew carrying a raspberry and a knife, wearing an embroidered habbit; against the pink pastel backdrop the meme text "i am plagued by concepts" is written in black sanserif font](../images/plaged_by_concepts.png)

As someone who lives FAIR, linked data, but knows most consumers of data neither understand nor care about it, what should I do? The answer isn't to avoid these concepts - it's to represent them responsibly until someone with actual authority shows up.

## Procrastination by way of blank nodes

My solution is deterministic blank nodes. Instead of coining URIs for concepts I don't own, I generate consistent blank nodes that can be reconciled later when authoritative sources emerge. This keeps my API stable while avoiding coining URIs I may eventually regret. Let me explain.

Previously I would have attempted to coin URIs for all my concepts, either at the dataset or higher level scope. For example, capturing the concept of running surface water from a river. In the source data for the API I have a table with a key and a label, the key acts as a notation.

```json
// You have no authority here, Jackie Weaver
{
  "@id": "http://environment.data.gov.uk/id/sample-material/2AZZ",
  "@type": ["skos:Concept", "sosa:FeatureOfInterest"],
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:notation": "2AZZ"
}
```

The issue is I currently don't have responsibility of the concept scheme for sample materials, and it's also not online. I know all the values, and I have a copy of it to make the service work but it's not within the scope of delivery for the water quality API. So instead of speaking with authority I've shifted to getting it down in code first to serve it via the API. How about as a blank node?

```json
// Procrastinating via blank nodes
{
  "@id": "_:sampleMaterial-2AZZ",
  "@type": ["skos:Concept", "sosa:FeatureOfInterest"],
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:notation": "2AZZ"
}
```

The key here isn't just using any blank node - it's using a **deterministic** blank node identifier. By concatenating the concept scheme name with the notation (`_:sampleMaterial-2AZZ`), I ensure that every time this concept appears in my API responses, it gets the same blank node identifier. 

> **Note**: This isn't standard RDF blank node syntax - it's my deterministic generation pattern from my source data. When serialized to actual RDF formats, these become proper blank nodes, but the consistent string ensures they all resolve to the same node across serializations. This isn't just semantic pedantry - it has real practical benefits.

When someone downloads multiple API responses and converts them to Turtle or N-Triples, all instances of `_:sampleMaterial-2AZZ` will be recognized as the same entity. Without this deterministic approach, you'd end up with multiple disconnected blank nodes for what should be the same concept, creating an unforgivable mess.

Here's what this looks like in practice - a real API response converted to Turtle:

```bash
curl -sSL --fail 'http://localhost:8000/sampling-point/53130070/sample?skip=0&limit=3&sampleMaterialType=2AZZ&complianceOnly=false' | rdfpipe -i json-ld -o ttl -
```

```ttl
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix hydra: <http://www.w3.org/ns/hydra/core#> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix sosa1: <http://www.w3.org/ns/sosa#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

<http://localhost:8000/sampling-point/53130070/sampling/1506412> a sosa1:Sampling ;
    dcterms:type _:samplingPurpose-CA ;
    sosa1:hasFeatureOfInterest <http://localhost:8000/sampling-point/53130070> ;
    sosa1:hasResult <http://localhost:8000/sampling-point/53130070/sample/1506412> ;
    sosa1:resultTime "2001-08-08"^^xsd:date ;
    sosa1:startTime "2000-08-18T12:20:00"^^xsd:dateTime .

<http://localhost:8000/sampling-point/53130070/sampling/1510110> a sosa1:Sampling ;
    dcterms:type _:samplingPurpose-CA ;
    sosa1:hasFeatureOfInterest <http://localhost:8000/sampling-point/53130070> ;
    sosa1:hasResult <http://localhost:8000/sampling-point/53130070/sample/1510110> ;
    sosa1:resultTime "2000-10-05"^^xsd:date ;
    sosa1:startTime "2000-09-20T12:00:00"^^xsd:dateTime .

<http://localhost:8000/sampling-point/53130070/sampling/2303318> a sosa1:Sampling ;
    dcterms:type _:samplingPurpose-CA ;
    sosa1:hasFeatureOfInterest <http://localhost:8000/sampling-point/53130070> ;
    sosa1:hasResult <http://localhost:8000/sampling-point/53130070/sample/2303318> ;
    sosa1:resultTime "2001-06-07"^^xsd:date ;
    sosa1:startTime "2000-11-29T00:01:00"^^xsd:dateTime .

<http://localhost:8000/sampling-point/53130070/sample/1506412> a sosa1:Sample ;
    sosa1:isResultOf <http://localhost:8000/sampling-point/53130070/sampling/1506412> ;
    sosa1:isSampleOf _:sampleMaterial-2AZZ,
        <http://localhost:8000/sampling-point/53130070> .

<http://localhost:8000/sampling-point/53130070/sample/1510110> a sosa1:Sample ;
    sosa1:isResultOf <http://localhost:8000/sampling-point/53130070/sampling/1510110> ;
    sosa1:isSampleOf _:sampleMaterial-2AZZ,
        <http://localhost:8000/sampling-point/53130070> .

<http://localhost:8000/sampling-point/53130070/sample/2303318> a sosa1:Sample ;
    sosa1:isResultOf <http://localhost:8000/sampling-point/53130070/sampling/2303318> ;
    sosa1:isSampleOf _:sampleMaterial-2AZZ,
        <http://localhost:8000/sampling-point/53130070> .

[] a hydra:Collection ;
    hydra:member <http://localhost:8000/sampling-point/53130070/sample/1506412>,
        <http://localhost:8000/sampling-point/53130070/sample/1510110>,
        <http://localhost:8000/sampling-point/53130070/sample/2303318> ;
    hydra:totalItems 129 ;
    hydra:view [ hydra:first <http://localhost:8000/sampling-point/53130070/sample?skip=0&limit=3&sampleMaterialType=2AZZ&complianceOnly=false> ;
            hydra:last <http://localhost:8000/sampling-point/53130070/sample?skip=126&limit=3&sampleMaterialType=2AZZ&complianceOnly=false> ;
            hydra:next <http://localhost:8000/sampling-point/53130070/sample?skip=3&limit=3&sampleMaterialType=2AZZ&complianceOnly=false> ] .

_:sampleMaterial-2AZZ a skos:Concept,
        sosa1:FeatureOfInterest ;
    skos:notation "2AZZ" ;
    skos:prefLabel "RIVER / RUNNING SURFACE WATER" .

_:samplingPurpose-CA a skos:Concept ;
    skos:notation "CA" ;
    skos:prefLabel "COMPLIANCE AUDIT (PERMIT)" .
```

Notice how `_:sampleMaterial-2AZZ` appears once in the graph but is referenced by multiple samples - exactly what we want. 

## When the kettles come out: reconciliation without regret

The beauty of this approach is that when the authoritative concept scheme eventually goes online (and it will, because I'm also building that service), I can simply add reconciliation triples without breaking anything. This is where semantic versioning becomes your friend - adding triples is a patch-level change at most. It neither changes the shape of the API's JSON, nor previously coined URIs.

```json
// Future state - same identifier, now with authority
{
  "@id": "_:sampleMaterial-2AZZ",
  "@type": ["skos:Concept", "sosa:FeatureOfInterest"],
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:notation": "2AZZ",
  "skos:exactMatch": "http://environment.data.gov.uk/def/sample-material/2AZZ",
  "rdfs:definedBy": "http://environment.data.gov.uk/def/sample-material/"
}
```

Now I can fire up those kettles I avoided earlier. The blank node stays the same, existing API consumers continue to work, but new consumers can follow the `skos:exactMatch` to the authoritative source. Cool URIs don't change, and neither will these deterministic blank nodes.

This approach scales beautifully across different concept schemes. Whether it's determinands that eventually align with QUDT vocabularies, geographic regions that get proper Ordnance Survey URIs, or measurement units that find their way into authoritative registries - the pattern remains the same. Add the reconciliation triples when you have them, leave the blank nodes as stable anchors within the service.

```json
// And it even supports multiple reconciliation targets
{
  "@id": "_:sampleMaterial-2AZZ",
  "@type": ["skos:Concept", "sosa:FeatureOfInterest"],
  "skos:prefLabel": "RIVER / RUNNING SURFACE WATER",
  "skos:notation": "2AZZ",
  "skos:exactMatch": "http://environment.data.gov.uk/def/sample-material/2AZZ",
  "rdfs:definedBy": "http://environment.data.gov.uk/def/sample-material/",
  "skos:closeMatch": "http://purl.obolibrary.org/obo/ENVO_00000022"
}
```

In a perfect world, every concept would have an authoritative URI from day one. In the real world, sometimes the most responsible thing you can do is admit you're not the authority - yet. Deterministic blank nodes let you build useful services today while keeping the door open for proper reconciliation tomorrow. It's procrastination with a purpose.
