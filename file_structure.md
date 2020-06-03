# Package File Structure

Underlay packages are represented locally as a file hierarchy under a `.underlay` directory, similar to `node_modules` or `.git`.

## Table of Contents

- [Package-level assertions](#package-level-assertions)
- [Package state](#package-state)
- [Ground-level assertions](#ground-level-assertions)
- [Ground-level files](#ground-level-files)
- [Subpackages](#subpackages)

## Package-level assertions

> `.underlay/.package`

A package's source of truth is the `.underlay/package` folder.

Assertions are the fundamental unit of data in the Underlay; rather than carrying a snapshot of a dataset's state, packages carry a log of assertions representing its incremental evolution. Given this log, anyone can construct a materialized "state" by _reducing_ over the assertions with respect to a schema.

The Underlay also uses the same approach - reducing assertions into a state - to represent and manage packages themselves. This means that packages aren't defined with a manifest or `package.json` file; instead, packages are defined by the `.underlay/package` _folder_. Inside this folder are JSON-LD "package assertions", each describing a positive, declarative update to the package, like adding a resource or setting the description. These package assertions are reduced over the fixed schema `package.shex` to produce the package state in the `.underlay/index.jsonld` file.

We use "package level" to refer to these lower meta-assertions, and "ground level" to refer to the package as exposed to users (for whom the contents of `.underlay/package` hide in the basement). Canonically, package-level assertions are named `[timestamp].[cid].jsonld`, where `[timestamp]` is the value of the assertion's date property in milliseconds since Unix epoch, and `[cid]` is the base-32 CIDv1 of the assertion.

For example, a package made up of three package assertions might have a file structure like:

```
- .underlay/
  - .package/
    - 1590077244008.bafkreie5o35s2los3qbutiy22gtb2ejd5tkmicc7ddjsfc3yairxmcbfyy.jsonld
    - 1590077968922.bafkreiff5vhxqsw3o6ovvgywww2etkld2h6muvip7gwzsxcr4eu6p4kbou.jsonld
    - 1590246664266.bafkreifpmzrev2mvtgc5afm4mwuozjbym6nvsohs7f6ejh2vt77x4gdq5q.jsonld
```

Packages, the package schema, and package assertions are described in more detail [here](package_schema.md).

This single level of recursion automatically unlocks some powerful features. Users might want to know both what a package says was true at a certain time ("John lived in Reno in 1990") and _also_ how that value changed throughout a package's history ("We later find out from a more credible source that he actually lived in Carson City"). In databases this is called bitemporal modelling; here it follows naturally from the package representation.

These package-level assertions also serve as the semantic grounding for various schema annotations. Schema annotations are heuristics for selecting values to populate cardinality-constrained properties - some heuristics like "greatest numeric value" only depend on the values themselves, but others like "last write wins" depend on meta-level access to source assertions.

In order to achieve this, the package schema `package.shex` that the package-level assertions are reduced over cannot reference any meta annotations itself. Just like every recursive algorithm must have a base case, the package assertions must be self-evident (including being unordered). This is why `package.shex` uses `rex:sort rex:latest` and `rex:with dcterms:date` annotations instead of `rex:lrw`: these assertions serve as the _basis_ for ground-level `rex:lrw` annotations.

## Package state

> `.underlay/index.jsonld`

The contents of `.underlay/package` are reduced over `package.shex` to construct the package state; this result is framed to produce a JSON-LD document which is written to `.underlay/index.jsonld`. This is what gets served in the REST API to GET requests at package paths.

Packages and package assertions are described in more detail [here](packages.md).

## Ground-level assertions

> `.underlay/*.jsonld`

After the package assertions are reduced to construct the package state, assertion resources found in the package state are retrieved (using IPFS if necessary) and copied to the `.underlay/` folder using their filenames.

In a package, each assertion is either _named_ with a resource URI that appends a single path element to the package's resource URI, or _unnamed_ and is referenced only by hash. A named assertion's filename is just the last path element, which will canonically end with the `.jsonld` file extension. An unnamed assertion's filename is its base-32 CIDv1 plus the `.jsonld` file extension.

For example, in a package with resource URI `http://example.com/hello` with one assertion named `http://example.com/hello/a.jsonld` and another unnamed assertion, the `.underlay` folder might look like

```
- .underlay/
  - .package/
    - 1590077244008.bafkreie5o35s2los3qbutiy22gtb2ejd5tkmicc7ddjsfc3yairxmcbfyy.jsonld
    - 1590077968922.bafkreiff5vhxqsw3o6ovvgywww2etkld2h6muvip7gwzsxcr4eu6p4kbou.jsonld
    - 1590246664266.bafkreifpmzrev2mvtgc5afm4mwuozjbym6nvsohs7f6ejh2vt77x4gdq5q.jsonld
  - a.jsonld
  - bafkreifzhkqugn4qvuc5hio7n6eim7ecw5etf7ayahhn2nzs5cqlda36ri.jsonld
```

## Ground-level files

> `.underlay/*.*`

Files act just like assertions, except unnamed files have no file extension in their filename (even though their MIME type is explicitly known by the package). Adding a named file `http://example.com/hello/b.csv` and an unnamed file to the package would result in a folder structure like

```
- .underlay/
  - .package/
    - 1590077244008.bafkreie5o35s2los3qbutiy22gtb2ejd5tkmicc7ddjsfc3yairxmcbfyy.jsonld
    - 1590077968922.bafkreiff5vhxqsw3o6ovvgywww2etkld2h6muvip7gwzsxcr4eu6p4kbou.jsonld
    - 1590246664266.bafkreifpmzrev2mvtgc5afm4mwuozjbym6nvsohs7f6ejh2vt77x4gdq5q.jsonld
  - a.jsonld
  - b.csv
  - bafkreifzhkqugn4qvuc5hio7n6eim7ecw5etf7ayahhn2nzs5cqlda36ri.jsonld
  - bafyreiesa36vuzxe2ok5lxm5rj7ozgltk7cb4qxnn4yvynijsyek6326fe
```

## Subpackages

> `.underlay/*/*`

Assertions and files are referenced by their content-hash, while subpackages are referenced by the hash of their own `.package` folder - in other words, a package "version" is just a specific set of package assertions.

After the package assertions are reduced to construct the package state, subpackage references are retrieved (again using IPFS) to create `[name]/.package` folders, where `[name]` is the last path element of the subpackage's resource URI. These subpackages are recursively expanded: their own package-level assertions get reduced, their own ground-level assertions are retrieved, and so on.

For example, if we add as subpackages `http://example.com/hello/foo` and `http://pets.com/bar`, our folder structure would look something like

```
- .underlay/
  - .package/
    - 1590077244008.bafkreie5o35s2los3qbutiy22gtb2ejd5tkmicc7ddjsfc3yairxmcbfyy.jsonld
    - 1590077968922.bafkreiff5vhxqsw3o6ovvgywww2etkld2h6muvip7gwzsxcr4eu6p4kbou.jsonld
    - 1590246664266.bafkreifpmzrev2mvtgc5afm4mwuozjbym6nvsohs7f6ejh2vt77x4gdq5q.jsonld
  - foo/
    - .package/
    - foo-assertion-1.jsonld
    - foo-file-1.csv
  - bar/
    - .package/
    - baz/
      - .package/
    - bar-assertion-1.jsonld
    - bar-file-1.pdf
  - a.jsonld
  - b.csv
  - bafkreifzhkqugn4qvuc5hio7n6eim7ecw5etf7ayahhn2nzs5cqlda36ri.jsonld
  - bafyreiesa36vuzxe2ok5lxm5rj7ozgltk7cb4qxnn4yvynijsyek6326fe
```
