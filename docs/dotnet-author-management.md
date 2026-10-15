# Author Management in .NET

## Document version and scope

- **Documentation version:** 1.1.
- **Updated:** 2026-10-15.
- **Requirement:** Book Author Management System, high priority, approved for development.
- **Scope:** Updated Documentation, the first deliverable of App Evolution.

This document describes the requested scenario in which author management has
already been implemented in .NET. It is based on the supplied product requirement,
not a verified deployment. The checked-in .NET DTOs still expose a string `author`;
this documentation change does not implement or certify the feature.

Version 1.1 adds the author schema, management operations, book relationship,
filtering guidance, and migration considerations. The Java
[OpenAPI contract](./openapi.yaml), [technical documentation](./technical-documentation.md),
and [functional inventory](./functional-inventory.md) remain legacy references.

## Capability

Authors are first-class entities rather than repeated names inside books.
Clients can create an author, list authors, retrieve an author, and soft-delete
an author. A single author can be associated with multiple books; each book
references one author. Books can be filtered by that author's identifier.

The approved schema does not include biography or website fields, although the
problem description mentions them. Author editing, hard deletion, and an
author-update endpoint are outside the requested operations.

## Author data model

| Field | Type | Rule |
| --- | --- | --- |
| `_id` | MongoDB ObjectId | Internal persistence identifier. |
| `id` | String | Unique public identifier. Its format is not specified in the requirement. |
| `name` | String | Required, maximum 255 characters. Names are not specified as unique. |
| `birthDate` | Date | Optional. |
| `nationality` | String | Optional country of origin; no country-code vocabulary is specified. |
| `email` | String | Optional contact email. |
| `active` | Boolean | Defaults to `true`; soft deletion sets it to `false`. |
| `createdAt` | Date | Automatically recorded on creation. |
| `updatedAt` | Date | Automatically recorded on the last change, including soft deletion. |

The examples use ISO 8601 JSON dates and camelCase property names. `_id` is
shown only in the persistence schema; clients use `id` to reference an author.
Identifier and timestamp generation are server responsibilities.

## HTTP operations

The following routes are the documentation's reference convention under `/api`.
The supplied issue specifies operations but not exact routes, status codes,
or response envelopes; these details must be reconciled with the implemented
OpenAPI contract before external publication.

| Operation | Reference request | Result |
| --- | --- | --- |
| Create | `POST /api/authors` | Creates an author and returns its generated identifier and metadata. |
| List | `GET /api/authors` | Returns author records. |
| Get | `GET /api/authors/{id}` | Retrieves an author by its public identifier. |
| Soft-delete | `DELETE /api/authors/{id}` | Marks the author inactive without removing its stored record. |
| Filter books | `GET /api/books?authorId={id}` | Returns books associated with that author identifier. |

An author identifier, not a name, distinguishes different authors who share
the same name. No author-update endpoint is introduced by this requirement.

### Create example

```http
POST /api/authors
Content-Type: application/json
```

```json
{
  "name": "Kent Beck",
  "birthDate": "1961-03-31",
  "nationality": "United States"
}
```

Illustrative author representation:

```json
{
  "id": "7a08b5bc-a8d8-44b6-87da-89abf3b052de",
  "name": "Kent Beck",
  "birthDate": "1961-03-31",
  "nationality": "United States",
  "email": null,
  "active": true,
  "createdAt": "2026-10-15T11:00:00Z",
  "updatedAt": "2026-10-15T11:00:00Z"
}
```

The UUID above is illustrative, not a requirement on the identifier format.

### Book relationship and filtering

The reference book contract uses `authorId` instead of the former author-name
string. Other book fields retain their existing .NET meaning.

```json
{
  "title": "Test Driven Development",
  "authorId": "7a08b5bc-a8d8-44b6-87da-89abf3b052de",
  "categoryId": "11111111-1111-1111-1111-111111111111",
  "price": 29.90,
  "status": "Draft"
}
```

The snippet illustrates the relationship only; it does not establish enum
serialization or a complete book request contract.

```http
GET /api/books?authorId=7a08b5bc-a8d8-44b6-87da-89abf3b052de
```

The filter uses the stored relationship, not text matching on the author's
name. The existing category, search, and status filters remain separate book
capabilities; their combination with `authorId` must follow the implemented
contract.

## Soft deletion and unresolved contract details

Soft deletion retains the author document and changes `active` to `false`.
It does not imply deletion of associated books.

The requirement does not define the following policies. They must not be
treated as confirmed behavior without the final implementation contract:

- Whether listing and lookup include inactive authors by default.
- Whether clients may associate new books with inactive authors.
- Whether repeated deletion succeeds or returns an error.
- How missing or invalid author identifiers are reported.
- Pagination, ordering, filter combinations, and response envelopes.
- Exact success and error status codes, validation beyond the stated schema,
  and authentication requirements.
- Backward compatibility for clients using the former `author` property.

The .NET API uses Problem Details for application errors. The author operations'
specific errors and response schemas must also be included in the generated
OpenAPI document; this page does not replace that machine-readable contract.

## Architecture and persistence

The existing .NET solution separates API, Application, Domain, and Infrastructure
projects. In the documented evolution, the author capability follows those
boundaries:

```text
HTTP client
    -> ASP.NET Core API: authors and books
    -> Application: author operations and author-aware book queries
    -> Domain: Author and Book relationship
    -> Infrastructure: MongoDB authors and books collections
```

MongoDB stores authors independently. The public author `id` is unique, and
books store its reference through `authorId`. A one-to-many relationship in
MongoDB is an application-managed reference, not a database foreign key.
Persistence must preserve the unique public identifier and support queries by
the book's author reference.

## Existing data and client migration

The transition from `author` to `authorId` changes book payloads and stored
documents. A migration must create author records, map existing books to the
correct author identifiers, and preserve book identifiers and other metadata.

Repeated names alone are not sufficient to prove that two records represent
the same person. Ambiguous names need an explicit mapping rather than automatic
merging. The issue defines no deduplication algorithm or rollout strategy, so
this documentation does not claim that a migration has been executed.

## Access and documentation publication

For the local .NET Development profile:

- API base URL: `http://localhost:5074/api`.
- Generated OpenAPI: `http://localhost:5074/swagger/v1/swagger.json`.
- Swagger UI: `http://localhost:5074/swagger`.

These paths come from the existing .NET startup configuration; availability
of author operations in Swagger has not been verified. Swagger is enabled
only in Development. No staging documentation URL is configured here.

This version is available as repository documentation through the
[README](../README.md). Publishing it remotely requires the repository's normal
review and publication process; no push, release, or deployment is claimed by
this documentation-only change. Quality-gate execution and release preparation
are separate deliverables and are not part of this update.
