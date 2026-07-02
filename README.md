# Luminair CMS - Core Documentation

Luminair is a Schema-Driven CMS platform inspired by Strapi, built for Speed and Reliability. 

## Key Differences from Strapi

* **Field-Level Internationalization (i18n)**: Unlike Strapi (which duplicates the entire document per locale), Luminair configures localization at the individual attribute/field level. Single documents store multi-lingual text dynamically (stored as `jsonb` objects in the database).
* **Hexagonal & Domain-Driven Design**: The system is split into distinct domain and infrastructure layers to maximize testability and maintainability.
* **Stable UUID Identifiers**: Documents are identified by stable UUIDs (`document_id`) across drafts, modifications, and publications.

## Shared Documentation Index

These documents serve as the shared specifications for both the frontend React administration panel and the backend Rust service:

* **[REST API](api.md)**: REST API endpoint references, request payload schemas, sorting & filtering conventions, and relation management rules.
* **[Draft & Publish Workflow](draft-publish.md)**: Lifecycle states and transitions (draft, published, modified).
