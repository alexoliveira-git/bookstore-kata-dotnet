# Technical documentation

## Documentation scope

This document records the **Java legacy implementation**. Its limitations,
including the unsupported author filter, do not describe the assumed .NET
author-management evolution. See [Author management in .NET](./dotnet-author-management.md)
for documentation version 1.1 (2026-10-15).

## System overview

Bookstore is a Java 11 HTTP API backed by MongoDB. Jersey provides the JAX-RS
resource layer, Grizzly hosts the HTTP server, JSON-B serializes request and
response objects, and the synchronous MongoDB Java driver handles persistence.
The application is a single service; it has no authentication, external service
integrations, or message broker.

```text
HTTP client
    │ JSON over HTTP
    ▼
Grizzly (:8080, base path /api)
    ▼
Jersey BookController (/books)
    ▼
BookService ── MongoDB sync driver ── MongoDB (:27017)
    │
    └── Book / Category models
```

The endpoint contract, payloads, and response statuses are documented in the
[OpenAPI specification](./openapi.yaml).

## Project structure

```text
src/
  main/java/com/bookstore/
    BookstoreApp.java                 Server startup and Jersey registration
    controller/
      BookController.java             HTTP routes and request-to-service mapping
      CreateBookDTO.java               Book creation request
      UpdateBookDTO.java               Book update request
    model/
      Book.java                        Book API/domain representation
      Category.java                    Static category list and category model
    service/
      BookService.java                 MongoDB access and book operations
  test/java/com/bookstore/controller/
    BookControllerTest.java            Controller unit test
mongo-init/
  01-init-bookstore.js                 MongoDB category collection seed
docs/
  openapi.yaml                         HTTP API contract
  technical-documentation.md           This document
Dockerfile                             Multi-stage Java build and runtime image
docker-compose.yml                     Local API and MongoDB services
pom.xml                                Maven dependencies and build plugins
```

## Runtime and integrations

### HTTP service

`BookstoreApp` starts Grizzly on `0.0.0.0:8080` with `/api` as the base URI,
registering JSON-B and `BookController` with Jersey. The controller exposes
health, list, lookup, create, and update operations under `/books`. It maps
requests to `BookService` and returns JAX-RS responses; business and database
operations are synchronous.

### MongoDB

`BookService` connects to `mongodb://mongodb:27017`, uses database `bookstore`
and collection `books`. It opens a Mongo client for each list or write
operation, maps MongoDB documents to `Book` objects, and closes the client
after operations. MongoDB generates the `books` collection when the first book
is inserted. Each document uses an ObjectId `_id`, book fields, and BSON dates
for `inactiveDate` and `publisherDate`.

Categories are currently defined in `Category.STATIC_CATEGORIES` (IDs 1–6).
Creation resolves the request's `categoryId` value as a category name and
persists its numeric ID. The MongoDB seed separately creates a `categories`
collection, but the application does not read that collection; the in-memory
list is authoritative for API operations.

`docker-compose.yml` sets `MONGODB_URI`, but the current service does not read
that environment variable: the connection URI is hardcoded in `BookService`.
Changing the MongoDB address therefore requires a code change. Compose exposes
MongoDB on host port 27017 and keeps its data in the `mongodb-data` volume.

### Data and operation behavior

- `Book` is both the service model and the JSON response shape; there is no
  separate persistence entity or mapping layer.
- Listing loads all documents, then filters known categories in application
  memory. Unknown category names result in an unfiltered list.
- Lookup by title loads the collection and compares titles exactly. ID lookup
  also scans the collection and currently compares strings by reference, which
  can fail for matching ID values.
- Creation inserts a new document with an ObjectId. An unknown category name
  can cause an exception because category validation returns `null`.
- Update attempts to set `_id`, `quantity`, `active`, and `inactiveDate` on a
  matching ObjectId. MongoDB treats `_id` as immutable, so this update can fail
  with a database error. It does not create missing documents; if the database
  accepts the operation, the HTTP layer returns 201 regardless of whether a
  document matched.
- The controller accepts an `author` lookup parameter but deliberately throws
  because author filtering is not implemented. There is no delete operation.

These describe current implementation details, not recommended API design.
See the OpenAPI document for externally observable behavior and limitations.

## Build, run, and test

Requirements: JDK 11, Maven, and Docker with Docker Compose (or a reachable
MongoDB host available at `mongodb:27017`).

```bash
# Run API and MongoDB
docker-compose up --build

# Run tests and package locally
mvn test
mvn package
```

The API listens on `http://localhost:8080/api`; MongoDB is available at
`mongodb://localhost:27017` from the host. The MongoDB initialization script
runs when the database volume is first created. The container image uses a
multi-stage Maven/JDK 11 build and a JRE 11 runtime; its package step skips
tests. Maven configures JUnit 5, Mockito, and JaCoCo, with coverage reports
generated during `mvn verify`; no coverage threshold is enforced in the
current build.

## Technical constraints

- MongoDB connection settings are hardcoded, and database operations create
  clients per operation.
- The server has no configured graceful-shutdown handling or readiness check.
- There is one controller unit test and no database integration tests.
- No application-level authentication, validation framework, or standardized
  error response format is configured.
