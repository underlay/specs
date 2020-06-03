# Package schema

```
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX u: <http://underlay.org/ns#>
PREFIX rex: <http://underlay.org/ns/rex#>

u:Package bnode {
  a [ u:Package ] ;
  dcterms:isVersionOf iri ;
  dcterms:date xsd:dateTime ;
  dcterms:title xsd:string // rex:ref dcterms:date ;
  dcterms:description xsd:string // rex:ref dcterms:date ;
  dcterms:subject xsd:string *  // rex:ref dcterms:date ;
  u:schema iri /^dweb:\/ipfs\/[a-z2-7]{59}$/ ? // rex:ref dcterms:date ;
  u:contains @_:assertion OR @_:file OR @_:package * // rex:ref dcterms:date ;
} // rex:key dcterms:isVersionOf

_:assertion bnode {
  a [ u:Assertion ] ;
  dcterms:isVersionOf iri ? ;
  dcterms:date xsd:dateTime // rex:ref rex:latest ;
  u:content iri /^ul:[a-z2-7]{59}$/ // rex:ref dcterms:date ;
} // rex:key dcterms:isVersionOf

_:file bnode {
  a [ u:File ] ;
  dcterms:isVersionOf iri ? ;
  dcterms:date xsd:dateTime ;
  dcterms:format xsd:string // rex:ref dcterms:date ;
  u:content iri /^dweb:\/ipfs\/[a-z2-7]{59}$/ // rex:ref dcterms:date ;
} // rex:key dcterms:isVersionOf

_:package bnode {
  a [ u:Package ] ;
  dcterms:isVersionOf iri ;
  dcterms:date xsd:dateTime ;
  u:content iri /^dweb:\/ipfs\/[a-z2-7]{59}$/ // rex:ref dcterms:date ;
} // rex:key dcterms:isVersionOf
```
