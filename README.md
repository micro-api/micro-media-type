[![Micro API](https://micro-api.github.io/micro-api/assets/logo.svg)](http://micro-api.org)

Micro API is a subset of [JSON-LD](http://json-ld.org) intended for representing hypermedia APIs. It includes a vocabulary, and semantics for [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations. As the name implies, it is intended to be concise and generic. Its registered media type is:

```yaml
Content-Type: application/vnd.micro+json
```

The current published version is **2017-04-25**, and the media type is [registered](https://www.iana.org/assignments/media-types/application/vnd.micro+json) with the [IANA](http://www.iana.org/).


## Introduction

Micro API simplifies [JSON-LD](http://json-ld.org) by limiting it to a subset which can be traversed reliably without using processing algorithms. Example payloads and HTTP requests should be considered non-normative.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this specification have the meaning defined in [RFC 2119](https://www.ietf.org/rfc/rfc2119).


## Vocabulary

*This section should be considered normative.*

Micro API introduces a small vocabulary focused on APIs that is not covered by other standards.

| Property | Type | Description |
|:---------|:-----|:------------|
| [`id`](http://micro-api.org/id) | `String`, `Number` | A unique value used for identifying resources, most often the primary key in a database. |
| [`meta`](http://micro-api.org/meta) | `Object` | Any meta-information may be contained here. |
| [`query`](http://micro-api.org/query) | `Object` | A container for showing information about the current query. |
| [`operate`](http://micro-api.org/operate) | `Object` | Reserved for application-specific, arbitrary operations to update resources. |
| [`error`](http://micro-api.org/error) | `Object` | If a request fails for any reason, it **MUST** return an error in the payload. |
| [`isArray`](http://micro-api.org/isArray) | `Boolean` | For properties, indicates if their values are arrays (this is lacking in RDF). If this is missing, it should be considered `false`. |


## Payload Restrictions

*This section should be considered normative.*

In general, the payload should look like the flattened form of JSON-LD, with some additional restrictions:

- The root node **MUST** be a singular object.
- There **MUST** be a top-level `@context` object, containing at least the context for Micro API: `http://micro-api.org/context.jsonld`, a `@base` and a `@vocab` valued by the entry IRI suffixed with a `#`.
- Resources **MUST** contain a unique `href` *and* `id`, no blank nodes are allowed. The `href` is an IRI (or an alias for `@id` in JSON-LD parlance), which is not to be confused with `id` which is an application-specific identifier.
- Resources **MUST** be represented as an array via the default `graph`.
- References **MUST** be represented as a singular object with either the `href` property *and/or* the `id` property.


## Entry Point

The expectation of a Micro API entry point is to enumerate types and properties and provide links to collections. It **MUST** contain the `definitions` at the top-level, valued as an array of objects, and the `type` as `Ontology`.

```http
GET /
```

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "type": "Ontology",
  "definitions": [
    { "href": "#name", "label": "Name",
      "propertyOf": [ "#Person", "#Movie" ], "propertyType": "xsd:string",
      "type": "Property", "comment": "Given name."
    },
    { "href": "#actor", "label": "Actors",
      "propertyOf": [ "#Movie" ], "propertyType": "#Person", "isArray": true,
      "type": "Property", "comment": "People who acted in a movie."
    },
    { "href": "#Person", "label": "Person",
      "type": "Class", "comment": "A human being."
    },
    { "href": "#Movie", "label": "Movie",
      "type": "Class", "comment": "A moving picture."
    }
  ],
  "Person": { "href": "/people" },
  "Movie": { "href": "/movies" }
}
```

The `@vocab` field of a Micro API **MUST** be the path to the API suffixed with the `#` character, so that dereferencing always refers to the entry point.


## Finding Resources

A `GET` request **MAY** be allowed on the collection IRI for a particular type.

```http
GET /movies
```

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "graph": [ {
    "type": "Movie",
    "href": "/movies/1",
    "id": 1,
    "name": "The Matrix",
    "actor": {
      "href": "/movies/1/actors",
      "id": [ 1, 2, 3 ]
    }
  } ]
}
```

Dereferencing a `href` **MUST** return types corresponding to that property.

```http
GET /movies/the-matrix/actors?limit=1
```

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "graph": [ {
    "type": "Person",
    "href": "/people/1",
    "id": 1,
    "name": "Keanu Reeves",
    "reverse": {
      "actor": {
        "href": "/people/1/acted-in",
        "id": [ 1 ]
      }
    }
  } ]
}
```


## Creating Resources

Requesting to create an resource **MAY** be allowed at the collection IRI for that type. The payload **MUST** be a valid Micro API document, and referenced IDs must be specified using the `id` property.

```http
POST /people
```

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "graph": [ {
    "type": "Person",
    "name": "John Doe",
    "reverse": {
      "actor": {
        "id": [ "memento" ]
      }
    }
  } ]
}
```

It may be helpful for the response to have a `Location` header, but it is not required since the response body may include a link to the created resource.


## Updating Resources

IDs **MUST** be specified in the payload per resource to update, and `PATCH` requests can be made wherever the resource exists (*corollary*: IDs can not be changed, only specified).

```http
PATCH /people
```

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "graph": [ {
    "type": "Person",
    "id": "john-doe",
    "name": "Johnny Doe",
    "reverse": {
      "actor": {
        "id": [ "point-break" ]
      }
    },
    "operate": {}
  } ]
}
```

If the a specified resource does not exist at the requested location, it **SHOULD** return an error. The assumption is that *the `PATCH` method replaces the fields specified*. There is a special `µ:operate` property which allows for arbitrary updates, which this specification is agnostic about. In common update cases, it may be desirable to reject upserts (the `PUT` method defines that [a resource may be created](http://greenbytes.de/tech/webdav/draft-ietf-httpbis-p2-semantics-21.html#PUT)), so `PATCH` is typically what you want to do.

`PATCH` requests can update existing resources, however Micro API does not define the semantics to create or delete resources with this method. By setting a link's `id` property to `null` (for a to-one relationship) or `[]` (empty array for a to-many relationship), it removes the link.


## Deleting Resources

```http
DELETE /people/john-doe
```

A delete request can return no payload (HTTP 204 No Content) if it succeeds. It can apply to any IRI, including collections.

```http
DELETE /people/john-doe/children
```

In this example, the request means delete all of the resources at this IRI, not just the link.


## Error Response

If a request fails for any reason, it **MUST** return a `error` object. The contents of the error object are opaque to this specification.

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "error": {
    "label": "NotFoundError",
    "comment": "The requested resource was not found."
  }
}
```


## Querying

Micro API does not specify anything about pagination, filtering, sparse fields, sorting, etc. For example, the `query` object **MAY** contain hints on what queries can be appended to GET requests, with further information about the query provided by a vocabulary (optional):

```json
{
  "@context": [
    "http://micro-api.org/context.jsonld",
    { "@base": "http://example.com/",
      "@vocab": "http://example.com/#"
    }
  ],
  "query": {
    "include": [],
    "sort": {},
    "field": {},
    "match": {},
    "limit": 1000,
    "offset": 0,
    "count": 0
  }
}
```


## Unregistered Media Type

One may use [MessagePack](http://msgpack.org) instead of JSON as the serialization format for greater bandwidth savings. Since MessagePack is an unregistered media type, the corresponding Micro API media type may be unregistered as well:

```yaml
Content-Type: application/x-micro-api
```

It is completely optional to support this unregistered media type, but it should be interpreted as Micro API with MessagePack enabled.


## Prior Art

Micro API builds upon [JSON-LD](https://www.w3.org/TR/json-ld/), which is a W3C recommendation. A JSON-based serialization format has the advantage of widespread tooling and developer understanding.

In contrast to [Linked Data Platform](https://www.w3.org/TR/ldp/), it does not use the Turtle format, which is useful only for working within [RDF](https://www.w3.org/RDF/). It also lacks a concept of "containers", which assumes that relationships are hierarchical. What is similar is that both stipulate which actions may be taken on resources.

Micro API is an alternative for [Hydra](http://www.markus-lanthaler.com/hydra/), another specification for Web APIs. It is much less prescriptive than Hydra, and is implicit in cases which Hydra is more explicit. For example, some differences are:

- **Collection**: all resources are collections.
- **Operation**: these are assumed to match HTTP semantics. Only `PATCH` requests may have special application-specific operations, using `µ:operate`.
- **Templated link**: clients must follow server links and only queries are allowed.
- **API documentation**: this is expected to contain natural language.

A key difference between Micro API and Hydra is that Micro API **does not** assume that documentation is machine-processable. Why this matters is that natural language may be the only way to express application logic.


## About

The source for this document is on [GitHub](https://github.com/micro-api/micro-api). It is licensed under the [CC0 1.0 License](https://raw.githubusercontent.com/micro-api/micro-api/master/LICENSE).
