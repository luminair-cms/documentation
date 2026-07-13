# REST API Documentation

This documentation describes the REST API exposed by the Luminair service. It is modeled after Strapi's REST API conventions while keeping project-specific behavior, especially around i18n, document IDs, and content-type handling.

> Rest API is modeled after same in Strapi. See: https://docs.strapi.io/cms/api/rest

## API structure

All endpoints are served under the `/api` prefix.

There are 2 types of routes:
- `meta` routes
- `documents` routes

### Meta routes

| Method | URL                          | Description                            |
| ------ | ---------------------------- | -------------------------------------- |
| GET    | `/api/meta/documents`        | List all registered document types     |
| GET    | `/api/meta/documents/:id`    | Get full schema for a document type    |

---

## Meta API Reference

The meta endpoints expose the **schema registry** — the set of content-type definitions that drive
every other part of the API. They are read-only and require no authentication in the current version.

---

### `GET /api/meta/documents`

Returns a summary list of every registered document type.

**Request:** no parameters.

**Response `200 OK`:**

```json
[
  {
    "id": "partners",
    "title": "Partners",
    "type": "collection",
    "description": "Partners with unique IDNO and Legal Entity"
  },
  {
    "id": "brands",
    "title": "Brands",
    "type": "collection",
    "description": "Brands of partners"
  }
]
```

> [!NOTE]
> The response is a plain JSON **array**, not wrapped in the standard `{ data, meta }` envelope
> that document CRUD endpoints use. This is intentional — meta endpoints carry no pagination.

#### `DocumentResponse` fields

| Field         | Type                        | Description                                    |
| ------------- | --------------------------- | ---------------------------------------------- |
| `id`          | String                      | The plural API identifier (used in all routes) |
| `title`       | String                      | Human-readable display name                    |
| `type`        | `"collection"` \| `"singleType"` | Document kind                             |
| `description` | String \| `null`            | Optional human-readable description            |

---

### `GET /api/meta/documents/:id`

Returns the full schema definition for a single document type, including all field and relation
attributes with their types, constraints, and options.

**Path parameter:** `:id` — the document type's plural API ID (e.g. `points-of-sale`).

**Response `200 OK`:**

```json
{
  "id": "points-of-sale",
  "title": "Points of sale",
  "type": "collection",
  "info": {
    "title": "Points of sale",
    "description": "Point of sale (concrete address)",
    "singularName": "point-of-sale",
    "pluralName": "points-of-sale"
  },
  "options": {
    "draftAndPublish": true,
    "localizations": ["en", "ro", "ru"]
  },
  "attributes": [
    {
      "id": "title",
      "type": "text",
      "unique": false,
      "required": false,
      "constraints": []
    },
    {
      "id": "latitude",
      "type": { "decimal": { "precision": 10, "scale": 8 } },
      "unique": false,
      "required": true,
      "constraints": []
    },
    {
      "id": "partner",
      "relation": "belongsToOne",
      "target": "partners"
    }
  ]
}
```

**Error responses:**

| Status | When |
| ------ | ---- |
| `404 Not Found` | No document type with the given `:id` is registered |
| `422 Unprocessable Entity` | The `:id` value fails internal ID validation (e.g. empty string) |

---

#### `DetailedDocumentResponse` fields

| Field        | Type                              | Description                          |
| ------------ | --------------------------------- | ------------------------------------ |
| `id`         | String                            | Plural API ID                        |
| `title`      | String                            | Display name                         |
| `type`       | `"collection"` \| `"singleType"` | Document kind                        |
| `info`       | `DocumentInfo`                    | Full naming metadata                 |
| `options`    | `DocumentOptions` \| `null`       | Feature flags; `null` when absent    |
| `attributes` | `Attribute[]`                     | Ordered list of fields and relations |

#### `DocumentInfo`

| Field          | Type             | Description                         |
| -------------- | ---------------- | ----------------------------------- |
| `title`        | String           | Display name (same as root `title`) |
| `description`  | String \| `null` | Optional description                |
| `singularName` | String           | Singular API ID (used for singletons) |
| `pluralName`   | String           | Plural API ID (used for collections) |

#### `DocumentOptions`

| Field            | Type     | Description                                                  |
| ---------------- | -------- | ------------------------------------------------------------ |
| `draftAndPublish`| Boolean  | Whether draft/publish workflow is enabled for this type      |
| `localizations`  | String[] | Enabled locale codes (e.g. `["en", "ro", "ru"]`); empty array when i18n is disabled |

---

#### `Attribute` — field vs. relation

Each entry in `attributes` is **one of two shapes**, discriminated by the presence of `type` (field)
or `relation` (relation). They are serialised as an untagged union — the client must check which
keys are present.

##### Field attribute shape

```json
{
  "id": "idno",
  "type": "<FieldType>",
  "unique": true,
  "required": true,
  "constraints": [ ... ]
}
```

| Field         | Type              | Description                                |
| ------------- | ----------------- | ------------------------------------------ |
| `id`          | String            | Attribute identifier (matches column name) |
| `type`        | `FieldType`       | See the FieldType reference below          |
| `unique`      | Boolean           | Whether a unique index is enforced         |
| `required`    | Boolean           | Whether `null` values are rejected         |
| `constraints` | `FieldConstraint[]` | Validation rules applied at write time   |

##### Relation attribute shape

```json
{
  "id": "partner",
  "relation": "belongsToOne",
  "target": "partners"
}
```

| Field      | Type           | Description                                         |
| ---------- | -------------- | --------------------------------------------------- |
| `id`       | String         | Attribute identifier                                |
| `relation` | `RelationType` | Directionality of the relation (see below)          |
| `target`   | String         | The plural API ID of the related document type      |

> [!IMPORTANT]
> Relation attributes do **not** carry `type`, `unique`, `required`, or `constraints` fields.
> Clients must not assume any of those keys exist when the `relation` key is present.

---

#### `FieldType` reference

The `type` field of a field attribute is a **tagged-union** value. Simple scalar types are
serialised as a plain string; parameterised types are serialised as a single-key object whose
value contains the parameters.

| Rust variant                          | JSON wire format                                 | Notes                              |
| ------------------------------------- | ------------------------------------------------ | ---------------------------------- |
| `Uid`                                 | `"uid"`                                          | Unique slug based on text          |
| `Uuid`                                | `"uuid"`                                         | UUID primary key                   |
| `Text`                                | `"text"`                                         | Plain string                       |
| `LocalizedText`                       | `"localizedText"`                                | Per-locale map `{ "en": "…" }`     |
| `Integer(Int16)`                      | `{ "integer": "int16" }`                         | 16-bit signed integer              |
| `Integer(Int32)` *(default)*          | `{ "integer": "int32" }`                         | 32-bit signed integer              |
| `Integer(Int64)`                      | `{ "integer": "int64" }`                         | 64-bit signed integer              |
| `Decimal { precision, scale }`        | `{ "decimal": { "precision": N, "scale": M } }`  | Fixed-point number                 |
| `Date`                                | `"date"`                                         | Calendar date (`YYYY-MM-DD`)       |
| `DateTime`                            | `"dateTime"`                                     | ISO-8601 timestamp                 |
| `Boolean`                             | `"boolean"`                                      | True/false                         |
| `Json`                                | `"json"`                                         | Arbitrary JSON blob                |

> [!IMPORTANT]
> Clients **must not** assume `type` is always a plain string. The `Integer` and `Decimal`
> variants are serialised as single-key objects. A robust client should check `typeof type`:
> if it is a string, treat it as a simple type name; if it is an object, read the first key as
> the type name and the value as its parameters.

---

#### `FieldConstraint` reference

Constraints are applied at document write time (create/update). Each constraint is serialised as
a **single-key object** whose key is the constraint name in camelCase.

| Constraint variant      | JSON wire format                      | Applicable types           |
| ----------------------- | ------------------------------------- | -------------------------- |
| `Pattern(String)`       | `{ "pattern": "^[0-9]{13}$" }`        | `text`, `uid`              |
| `MinimalLength(usize)`  | `{ "minimalLength": 4 }`              | `text`, `uid`, `localizedText` |
| `MaximalLength(usize)`  | `{ "maximalLength": 10 }`             | `text`, `uid`, `localizedText` |
| `MinimalIntegerValue(i32)` | `{ "minimalIntegerValue": 0 }`     | `integer`                  |
| `MaximalIntegerValue(i32)` | `{ "maximalIntegerValue": 100 }`   | `integer`                  |

An empty `constraints` array (`[]`) means no additional validation beyond nullability and
uniqueness is applied to the field.

---

#### `RelationType` reference

| Value            | Serialised as      | Owning side | Description                                                 |
| ---------------- | ------------------ | ----------- | ----------------------------------------------------------- |
| `HasOne`         | `"hasOne"`         | ✅ Yes       | This type owns a FK to exactly one instance of `target`     |
| `HasMany`        | `"hasMany"`        | ✅ Yes       | This type owns FKs to multiple instances of `target`        |
| `BelongsToOne`   | `"belongsToOne"`   | ❌ No (inverse) | Inverse of a `hasOne` or `hasMany` on the `target` type |
| `BelongsToMany`  | `"belongsToMany"`  | ❌ No (inverse) | Inverse of a `hasMany` on the `target` type             |

> [!NOTE]
> Only owning-side relations (`hasOne`, `hasMany`) can be managed via `connect`/`disconnect`
> in document write requests. Attempting to mutate an inverse relation returns `422 Unprocessable Entity`.



### Documents routes

- Collection routes: `/api/documents/:pluralApiId`
- Single-item routes: `/api/documents/:pluralApiId/:documentId`
- Singleton routes: `/api/documents/:singularApiId`

### Plural collection example

| Method | URL | Description |
| ------ | --- | ----------- |
| GET | `/api/documents/restaurants` | Get a list of restaurants |
| POST | `/api/documents/restaurants` | Create a restaurant |
| GET | `/api/documents/restaurants/:documentId` | Get a specific restaurant |
| PUT | `/api/documents/restaurants/:documentId` | Update a restaurant |
| DELETE | `/api/documents/restaurants/:documentId` | Delete a restaurant |

### Singleton example

> [!WARNING]
> Singletons are postponed to the post-MVP phase. The endpoints below are not implemented yet in this version of the service.

| Method | URL | Description |
| ------ | --- | ----------- |
| GET | `/api/documents/homepage` | Get the homepage content |
| PUT | `/api/documents/homepage` | Create or update the homepage content |
| DELETE | `/api/documents/homepage` | Delete the homepage content |

## Response structure

Responses follow the same structure as Strapi's REST API:

- `data` contains the requested content
- `meta` contains pagination or other metadata when applicable

### Collection response example

```json
{
  "data": [
    {
      "id": 2,
      "documentId": "8d0ef031-2a9a-4ea3-980f-e2a7f4803e95",
      "name": "BMK Paris Bamako",
      "description": {
        "en": "Description on English language",
        "ro": "Descriptie la limba Romaina",
        "ru": "Описание на Русском языке"
      },
      "createdAt": "2024-03-06T13:42:05.098Z",
      "updatedAt": "2024-03-06T13:42:05.098Z",
      "publishedAt": "2024-03-06T13:42:05.103Z"
    },
    {
      "id": 4,
      "documentId": "791620a6-1099-4a41-ad74-21c5a25ce9b2",
      "name": "Biscotte Restaurant",
      "description": [
        {
          "type": "paragraph",
          "children": [
            {
              "type": "text",
              "text": "Welcome to Biscotte restaurant! Restaurant Biscotte offers a cuisine based on fresh, quality products, often local, organic when possible, and always produced by passionate producers."
            }
          ]
        }
      ],
      "createdAt": "2024-03-06T13:43:30.172Z",
      "updatedAt": "2024-03-06T13:43:30.172Z",
      "publishedAt": "2024-03-06T13:42:05.175Z"
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 1,
      "total": 2
    },
    "defaultLocale": "en"
  }
}
```

### Single-item response example

```json
{
  "data": {
    "id": 6,
    "documentId": "791620a6-1099-4a41-ad74-21c5a25ce9b2",
    "name": "Biscotte Restaurant",
    "description": [
      {
        "type": "paragraph",
        "children": [
          {
            "type": "text",
            "text": "Welcome to Biscotte restaurant! Restaurant Biscotte offers a cuisine bassics, such as 4 Formaggi or Calzone, and our original creations such as Do Luigi or Nduja."
          }
        ]
      }
    ],
    "createdAt": "2024-02-27T10:19:04.953Z",
    "updatedAt": "2024-03-05T15:52:05.591Z",
    "publishedAt": "2024-03-05T15:52:05.600Z"
  },
  "meta": {
    "defaultLocale": "en"
  }
}
```

## Error Handling

The Luminair REST API follows standard-compliant **RFC 7807** and **RFC 9457** specifications to represent API error payloads, using the `application/problem+json` media type. 

All API errors return a standard JSON object containing the following fields:

| Field | Type | Description |
| --- | --- | --- |
| `type` | String | A URI reference that identifies the problem type. Defaults to `"about:blank"` when no specific reference is provided. |
| `title` | String | A short, human-readable summary of the problem type (e.g. the canonical HTTP reason phrase). |
| `status` | Integer | The HTTP status code generated by the origin server for this occurrence of the problem. |
| `detail` | String | A human-readable explanation specific to this occurrence of the problem. |
| `instance` | String (Optional) | A URI reference that identifies the specific occurrence of the problem. |

### Error Status Codes

The API maps internal business logic and validation errors to standard HTTP status codes:

#### 1. `404 Not Found`
Returned when a requested resource, document, document type, or relation does not exist.
- **Example payload:**
```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "The requested resource was not found"
}
```

#### 2. `422 Unprocessable Entity`
Returned when validation checks fail, or when trying to perform invalid operations (e.g. attempting to update a relation that is not an owning relation).
- **Example payload:**
```json
{
  "type": "about:blank",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Relation is not an owning relation: categories"
}
```

#### 3. `409 Conflict`
Returned when there is a state conflict with the current state of the server (e.g. unique constraint violations).
- **Example payload:**
```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "A document with the same slug already exists"
}
```

#### 4. `500 Internal Server Error`
Returned when an unexpected error occurs on the server side. For security and to prevent information leakage, detailed internal error messages are **hidden** from the client and instead recorded in the server logs.
- **Example payload:**
```json
{
  "type": "about:blank",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An internal server error occurred"
}
```

---

## i18n behavior

In this project, internationalization is configured per field rather than per document.
Each localized field is represented as an object whose keys are locale codes, for example `en`, `ro`, and `ru`.

### Example localized field

```json
"description": {
  "en": "Description on English language",
  "ro": "Descriptie la limba Romaina",
  "ru": "Описание на Русском языке"
}
```

This format is used for both read and write operations.

## Supported REST query parameters

The service supports query parameters similar to Strapi's REST API, including:

- `filters`: filter content by field values
- `sort`: specify result ordering
- `pagination`: control page and page size
- `fields`: select a subset of fields to return
- `populate`: include relational fields or nested content
- `locale`: select a locale when applicable

For more details on field selection and population syntax, see Strapi's REST docs: https://docs.strapi.io/cms/api/rest/populate-select

### Pagination

Use `pagination[page]` and `pagination[pageSize]` to paginate results.

Example:

```http
GET /api/restaurants?pagination[page]=2&pagination[pageSize]=10
```

### Sorting

Sort results using `sort`. You can sort by a single field, or by multiple fields using a comma-separated list.

Examples:
```http
GET /api/restaurants?sort=createdAt:desc
GET /api/restaurants?sort=name:asc,createdAt:desc
```

### Publication Status

Query documents based on their publication state using the `status` parameter. This is available for document types that have `draftAndPublish` enabled.

Supported values:

- `published` (default) — Returns only published document versions. This returns only published row at any level of relations
- `draft` — Returns the latest editorial state of the document and all populated relations — the draft row if unpublished changes exist, otherwise the published row.

Examples:

```http
GET /api/restaurants?status=published
GET /api/restaurants?status=draft
```

When `status` is not specified, only published documents are returned. For document instances that have both a published and a draft row, `status=draft` may return two versions for the same document ID.

### Filtering

Filter by field values using `filters`.
Common operators include `$eq`, `$ne`, `$contains`, `$lt`, `$gt`, and others supported by Strapi-style filtering.

Examples:

```http
GET /api/restaurants?filters[name][$contains]=Biscotte
GET /api/restaurants?filters[price][$gt]=10
GET /api/restaurants?filters[category][slug][$eq]=italian
```

### Field selection

Select only the fields you need by using `fields`.
This reduces payload size and improves performance.

Example:

```http
GET /api/restaurants?fields=name,description
```

If you are returning populated relations, use `populate` to load those related fields explicitly.

### Population

Populate relational fields and nested objects using `populate`.
The `populate` parameter controls which relations and nested content are returned with the response.

> [!WARNING]
> Nested population (deep populate, e.g. `populate[author][populate]=something`) is not supported in the MVP backend. Only single-level population of direct relations (e.g. `populate=author` or `populate=*`) is supported.

Examples:

```http
GET /api/restaurants?populate=*
GET /api/restaurants?populate=author
GET /api/restaurants?populate[author]=*
GET /api/restaurants?populate[author][fields]=name,email
GET /api/restaurants?populate[gallery][fields]=url,caption
```

Use `populate=*` to include all relations in the response. For large documents, prefer explicit population to keep the returned payload minimal.

### Filtering within population

You can apply filters to populated relations to limit which related records are returned.
This is useful when you only need a subset of nested documents or relation items.

Examples:

```http
GET /api/restaurants?populate[reviews]=*&filters[reviews][rating][$gte]=4
GET /api/restaurants?populate[author]=*&filters[author][status][$eq]=active
```

In this project, filters inside `populate` work together with top-level filters, allowing both primary document selection and nested relation filtering in a single request.

### Combined selection and population

You can combine `fields` and `populate` to return a limited set of top-level fields while still loading related data:

```http
GET /api/restaurants?fields=name,description&populate=author
```

By default, `fields` applies to top-level document attributes. Use the `populate` parameter to control nested relations and their fields.

## Request examples

### Create a document

```http
POST /api/restaurants
Content-Type: application/json

{
  "data": {
    "name": "New Restaurant",
    "description": {
      "en": "New English description",
      "ro": "Descriere nouă în română",
      "ru": "Новое описание на русском"
    },
    "categories": {
      "connect": ["8d0ef031-2a9a-4ea3-980f-e2a7f4803e95"]
    }
  }
}
```

### Update a document

```http
PUT /api/restaurants/791620a6-1099-4a41-ad74-21c5a25ce9b2
Content-Type: application/json

{
  "data": {
    "name": "Updated Restaurant Name",
    "description": {
      "en": "Updated English description",
      "ro": "Descriere actualizată în română",
      "ru": "Обновленное описание на русском"
    }
  }
}
```

### Delete a document

```http
DELETE /api/restaurants/791620a6-1099-4a41-ad74-21c5a25ce9b2
```

### Publish a document

The publish endpoint changes a document's publication state from `Draft` to `Published`. This operation is only applicable to document types that have `draftAndPublish` enabled in their schema.

```http
POST /api/documents/restaurants/791620a6-1099-4a41-ad74-21c5a25ce9b2/publish
```

**Success Response (200 OK):**

Returns the published document instance. The response `data` object includes the publication details (`publishedAt`, `publishedBy`, and `revision`) flattened alongside other document properties:

```json
{
  "data": {
    "id": 6,
    "documentId": "791620a6-1099-4a41-ad74-21c5a25ce9b2",
    "createdAt": "2024-02-27T10:19:04.953Z",
    "updatedAt": "2024-03-05T15:52:05.591Z",
    "publishedAt": "2024-03-05T15:52:05.600Z",
    "publishedBy": "admin_user",
    "version": 2,
    "revision": 1,
    "name": "Biscotte Restaurant",
    "description": {
      "en": "Welcome to Biscotte restaurant!"
    }
  }
}
```

## Managing Relations

Relations between content types can be managed through the REST API by passing `connect`, `disconnect`, or `set` parameters in the request body. In accordance with Strapi 5, these operations can be supplied during document creation (`POST`) to automatically establish initial relations, or during updates (`PUT`). These operations work for both single-entry relations and multi-relations (one-to-many, many-to-one, many-to-many, and many-way relations).

### Connect

The `connect` operation establishes new relations with existing documents. It performs a partial update, meaning existing relations are preserved and new ones are added.

**Syntax:** Both shorthand and longhand syntax are supported:

```json
{
  "data": {
    "categories": {
      "connect": ["8d0ef031-2a9a-4ea3-980f-e2a7f4803e95", "791620a6-1099-4a41-ad74-21c5a25ce9b2"]
    }
  }
}
```

Longhand syntax with document objects:

```json
{
  "data": {
    "categories": {
      "connect": [
        { "documentId": "z0y2x4w6v8u1t3s5r7q9onm" },
        { "documentId": "j9k8l7m6n5o4p3q2r1s0tuv" }
      ]
    }
  }
}
```

**Example request:**

```http
PUT /api/restaurants/a1b2c3d4e5f6g7h8i9j0klm
Content-Type: application/json

{
  "data": {
    "categories": {
      "connect": ["z0y2x4w6v8u1t3s5r7q9onm", "j9k8l7m6n5o4p3q2r1s0tuv"]
    }
  }
}
```

> **Note:** In Luminair, internationalization is configured per field, not per document. If you need to connect relations for a specific locale, include the locale information within the field-level object. For example, when a field has multiple locale variants, specify which locale the connection applies to at the field level.

> **Note on MVP limitations:** The MVP version does not support ordering of relations. Positional arguments (`before`, `after`, `start`, `end`) are not available in this release.

### Disconnect

The `disconnect` operation removes existing relations. It performs a partial update, meaning other relations are preserved and only the specified ones are removed.

**Syntax:** Both shorthand and longhand syntax are supported:

```json
{
  "data": {
    "categories": {
      "disconnect": ["8d0ef031-2a9a-4ea3-980f-e2a7f4803e95"]
    }
  }
}
```

Longhand syntax:

```json
{
  "data": {
    "categories": {
      "disconnect": [
        { "documentId": "z0y2x4w6v8u1t3s5r7q9onm" },
        { "documentId": "j9k8l7m6n5o4p3q2r1s0tuv" }
      ]
    }
  }
}
```

**Example request:**

```http
PUT /api/restaurants/a1b2c3d4e5f6g7h8i9j0klm
Content-Type: application/json

{
  "data": {
    "categories": {
      "disconnect": ["z0y2x4w6v8u1t3s5r7q9onm"]
    }
  }
}
```

### Set

The `set` operation replaces all existing relations with a new set. It performs a full update, meaning all previous relations are removed and replaced with the specified ones.

> [!WARNING]
> The `set` relation operation is not implemented yet in the MVP backend. Attempting to use this operation will return a `422 Unprocessable Entity` error.

**Syntax:** Both shorthand and longhand syntax are supported:

```json
{
  "data": {
    "categories": {
      "set": ["8d0ef031-2a9a-4ea3-980f-e2a7f4803e95", "791620a6-1099-4a41-ad74-21c5a25ce9b2"]
    }
  }
}
```

Longhand syntax:

```json
{
  "data": {
    "categories": {
      "set": [
        { "documentId": "z0y2x4w6v8u1t3s5r7q9onm" },
        { "documentId": "j9k8l7m6n5o4p3q2r1s0tuv" }
      ]
    }
  }
}
```

**Example request:**

```http
PUT /api/restaurants/a1b2c3d4e5f6g7h8i9j0klm
Content-Type: application/json

{
  "data": {
    "categories": {
      "set": ["z0y2x4w6v8u1t3s5r7q9onm", "j9k8l7m6n5o4p3q2r1s0tuv"]
    }
  }
}
```

### Combining connect and disconnect

You can combine `connect` and `disconnect` operations in a single request to perform both partial additions and removals:

```http
PUT /api/restaurants/8d0ef031-2a9a-4ea3-980f-e2a7f4803e95
Content-Type: application/json

{
  "data": {
    "categories": {
      "connect": ["8d0ef031-2a9a-4ea3-980f-e2a7f4803e95"],
      "disconnect": ["791620a6-1099-4a41-ad74-21c5a25ce9b2"]
    }
  }
}
```

> **Important:** `set` cannot be combined with `connect` or `disconnect`. Use `set` only when you want to completely replace all relations.

For more details on relation management, see the [Strapi REST API documentation on relations](https://docs.strapi.io/cms/api/rest/relations).

## Notes

- `documentId` is a UUID that identifies the document instance.
- For singleton content types, the endpoint uses the singular API ID.
- Collections use the plural API ID for routing.
- `publishedAt` and `updatedAt` timestamps are returned when publication and update information are available.
