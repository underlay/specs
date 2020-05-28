# Package Server HTTP API

A package server exposes a REST HTTP API for managing Underlay resources.

- _Assertions_ are arbitrary RDF datasets
- _Files_ are arbitrary byte arrays, with a single explicit MIME type and a known integer size in bytes
- _Packages_ are a kind of container analogous to a directory or a Git repository. Packages have metadata like schemas and provenance and contain assertions, files, and other packages.

Like all REST services, package servers implicitly distinguish between abstract _resources_ and concrete _representations_. A resource is a conceptual target identified by a URL; a representation of a resource is a physical materialization in some known format of a version of that resource at a point in time. The REST API interfaces between the two, resolving requests for resources into concrete representations.

Abstractly, an assertion is a set of graphs of relations between entities; concretely they are represented as either [N-Quads](https://www.w3.org/TR/n-quads/) files or [JSON-LD](https://www.w3.org/TR/json-ld11) documents. Similarly, a package is a resource that contains other resources. Package representations also use RDF; however, unlike an assertion, whose representation is an entire RDF dataset, a package representation is a specific blank node (called the _package subject_) within an RDF dataset (containing triples that describe the package metadata and contents using this package subject). 

Representing a package subject is challenging because RDF blank nodes are anonymous by design. Package servers solve this by leveraging the [Universal Dataset Normalization Algorithm](https://json-ld.github.io/normalization/spec/) (URDNA2015) published by the [W3C Credentials Community Group](https://www.w3.org/community/credentials/), which serializes an abstract RDF dataset to a normalized N-Quads representation. The normalized result is canonical in the sense that any two isomorphic datasets will map to exactly the same representation. Part of the algorithm involves assigning canonical labels to every blank node in the dataset, so package servers use this canonical blank node label as fragment identifiers in a `Link` header with `rel="self"` to unambiguously represent packages. In most cases, the link target will be `<#c14n0>`, since the package subject will typically be the only blank node in the dataset.

## Headers

### `Link`

In addition to identifying package subjects, package servers also use [`Link` headers](https://tools.ietf.org/html/rfc8288) to identify resource _types_. This is necessary because the formats of the different kinds of resources can overlap - for example, it's not possible without context to tell whether a JSON-LD document returned in the response body of a `GET` request is supposed to be a package, an assertion, or just a file with MIME type `application/ld+json`.

This missing context is represented with a single `Link` header field:
- `Link: <http://underlay.org/ns#File>; rel="type"` indicates that the identified resource is a file
- `Link: <http://underlay.org/ns#Assertion>; rel="type"` indicates that the identified resource is an assertion, represented by the accompanying dataset
- `Link: <http://underlay.org/ns#Package>; rel="type", <#c14n0>; rel="self"` indicates that the identified resource is a package, represented by the blank node with canonical label `_:c14n0` in the accompanying dataset. Note that the self link target does not include the `_:` prefix, and although it will not always be strictly `#c14n0`, it will always match `/^#c14n\d+$/`.

`Link` headers appear in both requests (`POST`, `PUT`, `DELETE`) and responses (`HEAD`, `GET`).

### `ETag`

Every instance of a resource representation has an associated [entity-tag](https://en.wikipedia.org/wiki/HTTP_ETag), an opaque identifier that can be re-used to make conditional requests.

Package servers use IPFS UnixFS file CIDv1s in base 32 for each resource representation as its opaque entity-tag. [UnixFS](https://docs.ipfs.io/concepts/file-systems) is the default [protocol buffer](https://github.com/ipfs/go-unixfs/blob/master/pb/unixfs.proto)-based format for files used in IPFS; [CIDs](https://github.com/multiformats/cid#human-readable-cids) are self-describing content-hash identifiers.

For representations of file resources, the file itself is used as the contents of the UnixFS file; for assertions and packages, the dataset representation is normalized using the URDNA2015 algorithm and the N-Quads string is used as the contents of the UnixFS file.

Package servers **use the non-default option `raw-leaves`** when creating UnixFS files.
UnixFS files are split by default into 262144-byte chunks, which are linked from intermediate nodes to form a [merkle tree](https://en.wikipedia.org/wiki/Merkle_tree). 

The entity-tag for a file can be computed using the [IPFS CLI tool](https://docs.ipfs.io/reference/cli/#ipfs-add) 
```
$ echo 'Hello World' > foo
$ ipfs add --only-hash --raw-leaves --chunker size-262144 --cid-version 1 foo
added bafkreigsvbhuxc3fbe36zd3tzwf6fr2k3vnjcg5gjxzhiwhnqiu5vackey foo
 12 B / 12 B [=========================================================] 100.00%
```

The resulting the `ETag` field for this file would be:
```
ETag: "bafkreigsvbhuxc3fbe36zd3tzwf6fr2k3vnjcg5gjxzhiwhnqiu5vackey"
```

Note that entity-tags, per [RFC 7232](https://tools.ietf.org/html/rfc7232#section-2.3), are wrapped in quotation marks.

Package servers do not use weak entity-tags.

Entity-tags are returned with successful `HEAD`, `GET`, `POST`, `PUT`, and `MKCOL` responses.

### `Last-Modified`

Modification date can be used as an alternative to entity-tags. The `Last-Modified` header field is the date that the given resource was last modified, represented as an [HTTP-date](https://tools.ietf.org/html/rfc7231#section-7.1.1.1).

The last modified date is returned with successful `HEAD`, `GET`, `POST`, `PUT`, and `MKCOL` responses.

### `If-Match`

The value of an `If-Match` field must be a base32 CIDv1 UnixFSv1 entity-tag as described above.

`If-Match` can optionally be given with `PUT` and `DELETE` requests; if present, the action (put or delete) will only be performed if the field value is strictly equal to the current representation's entity-tag.

### `If-Unmodified-Since`

The value of an `If-Unmodified-Since` field must be [HTTP-date](https://tools.ietf.org/html/rfc7231#section-7.1.1.1).

`If-Unmodified-Since` can optionally be given with `PUT` and `DELETE` requests; if present, the action (put or delete) will only be performed if the field value is later than or equal to the last modified date of the resource.

### `If-None-Match`

The value of an `If-None-Match` field must be a base32 CIDv1 UnixFSv1 entity-tag as described above.

`If-None-Match` can optionally be given with `GET` requests; if present, no content will be returned in the response body if the field value is strictly equal to the current representation's entity-tag.

### `If-Modified-Since`

The value of an `If-Modified-Since` field must be [HTTP-date](https://tools.ietf.org/html/rfc7231#section-7.1.1.1).

`If-Modified-Since` can optionally be given with `GET` requests; if present, no content will be returned in the response body if the field value is later than or equal to the last modified date of the resource.

### `Accept`

An `Accept` header can optionally be given with `GET` requests to select an RDF serialization for packages and assertions. If given, the field value must be either `application/n-quads` or `application/ld+json`; if not given, `application/n-quads` will be used as a default.

The `Accept` field is ignore for `GET` requests to file resources, since files are of a single fixed format. For packages and assertions, the response body will be a JSON-LD representation of the resource for `Accept: application/ld+json`, and will be the URDNA2015 normalized N-Quads string representation of the resource otherwise.

### `Content-Type`

A `Content-Type` header must be provided with every `PUT` and `POST` request. If the resource in the request body is a package or assertion, the value of the `Content-Type` field must be either `application/n-quads` or `application/ld+json`. If the resource is a file, the value of the `Content-Type` field is used as the MIME type of the file (i.e. it ends up materialized as a XSD string in the RDF representation of the parent package).

A `Content-Type` header is returned with every successful `GET` request unless the request returns `304 Not Modified` with no content. Additionally, every `HEAD` request to a file resource returns the file's MIME type in a `Content-Type` head in the response (which has no body). A `Content-Type` field is not returned with `HEAD` responses for assertions or packages.

### `Content-Length`

A `Content-Length` field is returned with every HTTP response with a value of the number of bytes in the response body, which may be 0. As a special case, a successful `HEAD` request to a file resource returns the file size in the `Content-Length` field, even though the response body is empty, consistent with the semantics of `HEAD`.

### `Location`

A `Location` field is included in responses to successful `POST` requests. The value of the `Location` field is the relative URL of the resource that was just created.

## Methods

### `GET`

A `GET` request to a resource returns a representation of that resource.

#### Request headers

- `Accept` (optional)
- `If-None-Match` (optional)
- `If-Modified-Since` (optional)

#### Request body

`GET` does not accept a request body.

#### Response headers

- `Link`
- `ETag`
- `Last-Modified`
- `Content-Type`
- `Content-Length`

#### Response body

A successful `GET` will return a representation of the requested resource in the response body.

#### Status codes

- `404 Not Found`
- `406 Not Acceptable`
- `304 Not Modified`
- `200 OK`

### HEAD

A `HEAD` request functions identical to a `GET` request, except it never returns content in the response body.

#### Request headers

- `Accept` (optional)
- `If-None-Match` (optional)
- `If-Modified-Since` (optional)

#### Request body

`HEAD` does not accept a request body.

#### Response headers

- `Link`
- `ETag`
- `Last-Modified`
- `Content-Type`
- `Content-Length`

`Content-Type` and `Content-Length` do no describe the HTTP response body, which is empty for all `HEAD` requests; they describe the resource identified by the given URL.

#### Response body

`HEAD` does not return a response body.

#### Status codes

- `404 Not Found`
- `406 Not Acceptable`
- `304 Not Modified`
- `200 OK`

### `POST`

A `POST` request takes an assertion or file in the request body and inserts it into the package identified by the URL. If the resource at the given path exists, but is not a package, then `405 Method Not Allowed` is returned.

#### Request headers

- `Link`: must be either:
	- `<http://underlay.org/ns#Assertion>; rel="type"`
	- `<http://underlay.org/ns#File>; rel="type"`
- `Content-Type`: if the attached resource is an assertion, then `Content-Type` must be either:
	- `application/n-quads`
	- `application/ld+json`

#### Request body

The body of a `POST` request must be a resource representation consistent with the given `Link` and `Content-Type` headers.

#### Response headers

- `ETag`
- `Last-Modified`
- `Location`

#### Response body

`POST` does not return a response body.

#### Status codes

- `400 Bad Request`
- `404 Not Found`
- `405 Method Not Allowed`
- `415 Unsupported Media Type`
- `201 Created`

### `PUT`

A `PUT` request replaces the resource at the given path with the resource in the request body, or inserts the request body as a new resource at the given path if it does not already exist. If the given path does not exist and the parent path is not a pre-existing package, then `409 Conflict` is returned.

#### Request headers

- `Link`
- `Content-Type`: if the attached resource is an assertion or a package, then `Content-Type` must be either:
	- `application/n-quads`
	- `application/ld+json`
- `If-Match` (optional)
- `If-Unmodified-Since` (optional)

#### Request body

The body of a `PUT` request must be a resource representation consistent with the given `Link` and `Content-Type` headers.

#### Response headers

- `ETag`
- `Last-Modified`

#### Response body

`PUT` does not return a response body.

#### Status codes

- `400 Bad Request`
- `409 Conflict`
- `415 Unsupported Media Type`
- `412 Precondition Failed`
- `204 No Content`

### `MKCOL`

An `MKCOL` request creates a new package at the given URL. If a resource already exists at the given path, `405 Method Not Allowed` is returned, and if a package does not already exist at the parent path, `409 Conflict` is returned as per [RFC 4918](http://www.webdav.org/specs/rfc4918.html#METHOD_MKCOL).

#### Request headers

`MKCOL` accepts no request headers.

#### Request body

`MKCOL` does not accept a request body.

#### Response headers

- `ETag`
- `Last-Modified`

#### Response body

`MKCOL` does not return a response body.

#### Status codes

- `405 Method Not Allowed`
- `409 Conflict`
- `201 Created`

### `DELETE`

A `DELETE` request deletes a resource at the given URL, removing it from the parent package. A `DELETE` request to the root path `/` will return `405 Method Not Allowed`.

#### Request headers

- `If-Match` (optional)
- `If-Unmodified-Since` (optional)

#### Request body

`DELETE` does not accept a request body.

#### Response headers

`DELETE` returns no response headers.

#### Response body

`DELETE` does not return a response body.

#### Status codes

- `400 Bad Request`
- `404 Not Found`
- `405 Method Not Allowed`
- `412 Precondition Failed`
- `204 No Content`
