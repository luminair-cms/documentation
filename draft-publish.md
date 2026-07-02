# Draft & Publish Workflow

## Overview

Luminair implements a draft-and-publish workflow for document types that have `draftAndPublish` enabled in their schema options. This design allows content editors to work on changes in a "Draft" status while serving the last approved "Published" version of the document to public users.

## Key Concepts

- **Draft Version**: The current working copy of the document. This is the version actively modified by editors.
- **Published Snapshot**: An immutable frozen copy of the document representing a version approved for public release.
- **version**: An incremental number tracking the total modifications (saves, edits, publications) made to the document.
- **revision**: A sequential publication count (1, 2, 3, ...) assigned to published snapshots.
- **status**: Indicates the current editorial workflow stage of the document:
  - `DRAFT` — The document has been created but has never been published.
  - `PUBLISHED` — The document is currently published, and there are no pending draft changes.
  - `MODIFIED` — The document has a published version, but an editor has saved new draft modifications that are not yet published.

---

## Document Lifecycle

The status and version numbers transition as follows during the document lifecycle:

| Action | Status | Version | Revision | Published At | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Create** | `DRAFT` | `1` | `0` | `null` | Initial draft created |
| **Edit** | `DRAFT` | `2` | `0` | `null` | Draft updated by editor |
| **Publish** | `PUBLISHED` | `3` | `1` | `timestamp` | First snapshot (Revision 1) created |
| **Edit** | `MODIFIED` | `4` | `1` | `null` | Draft updated; snapshot remains unchanged |
| **Edit** | `MODIFIED` | `5` | `1` | `null` | Further draft updates |
| **Publish** | `PUBLISHED` | `6` | `2` | `timestamp` | Second snapshot (Revision 2) created |

---

## API Considerations

The REST API serves content depending on the requested document status:

- **Public API** (`GET /api/documents/:pluralApiId?status=published` or `GET /api/documents/:pluralApiId/:id`): Reads from the snapshot tables, returning only the latest published version.
- **Editor API** (`GET /api/documents/:pluralApiId?status=draft`): Reads the working copy, returning the latest draft state (the draft changes if the document is DRAFT/MODIFIED, otherwise the published copy).
- **History API** (`GET /api/documents/:pluralApiId/:id/revisions`): Lists all historical snapshots (Note: **Not implemented in the MVP**).

---

## Schema Configuration

To enable the draft-and-publish workflow for a collection or single-type, set `draftAndPublish` to `true` in the model schema options:

```json
{
  "options": {
    "draftAndPublish": true
  }
}
```

When `draftAndPublish` is `false` or omitted, updates are automatically published immediately.