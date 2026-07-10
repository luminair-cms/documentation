# Big React Project Layout Best Practices

When scaling a React + TypeScript application, a flat file structure quickly degrades into high coupling and difficulty navigating the codebase. Below is the recommended layout architecture for the Luminair CMS admin panel, designed to support clean separation of concerns and modular growth.

---

## 1. Directory Structure Blueprint

```
luminair-ui/
├── src/
│   ├── api/             # Global API client instance (e.g., axios wrapper, interceptors)
│   ├── components/      # Common/shared UI elements (Button, Table, Modal, Input)
│   ├── features/        # Business logic split by vertical domains (e.g., content, schemas)
│   │   ├── content/
│   │   │   ├── components/  # Specific UI widgets used only in this feature
│   │   │   ├── hooks/       # Custom React Query or local helpers for this feature
│   │   │   ├── services/    # Feature-specific API requests
│   │   │   └── index.ts     # Public API/Exports for the content feature
│   ├── pages/           # High-level route views (combines layout + feature components)
│   ├── store/           # Global client-side stores (e.g., Zustand UI stores)
│   ├── theme/           # Design tokens, theme config, and styling configurations
│   ├── App.tsx          # Global providers, router routes definition
│   └── main.tsx         # React DOM mounting entry point
```

---

## 2. Key Directories & Best Practices

### A. Shared Components (`src/components/`)
*   **Purpose**: Design system building blocks and generic utility components.
*   **Rules**:
    *   Must be completely **decoupled from business logic** (no API calls, no domain-specific state).
    *   Receive configuration purely via standard React `props`.
    *   Examples: Custom button wrappers, localized calendar pickers, generic loading spinner widgets, table components.

### B. Domain Features (`src/features/`)
*   **Purpose**: The core business modules of the application, structured around specific product capabilities (e.g., dynamic content management, schema building, settings configuration).
*   **Rules**:
    *   Colocate related code together: hooks, types, APIs, and sub-components belong inside the feature folder.
    *   **Enforce clean boundaries**: Use an `index.ts` at the root of each feature to export only what is required externally (e.g., public components or hooks). Other features must not import from internal paths of a sibling feature (e.g., import from `../content` is allowed, but `../content/components/InternalWidget` is forbidden).
    *   Examples: `src/features/content/` handles listing documents, creating translations, and localized forms.

### C. Pages (`src/pages/`)
*   **Purpose**: The entry points for React Router routes.
*   **Rules**:
    *   Keep pages thin. A page should simply retrieve route parameters (like `id` or query strings), apply the main shell layout, and compose the necessary feature components.
    *   Avoid direct business logic, inline query hooks, or heavy styled code inside `pages`.
    *   Examples: `src/pages/DocumentEditPage.tsx` imports the `DocumentForm` component from `features/content`.

### D. Services / API (`src/api/` & `features/*/services/`)
*   **Purpose**: Communicating with backend HTTP services.
*   **Rules**:
    *   Maintain a centralized base client configuration (e.g., token interceptors, baseURL, error handling) in `src/api/client.ts`.
    *   Place feature-specific requests inside `src/features/[feature_name]/services/`.
    *   Use TanStack Query (React Query) hooks to manage loading states, caching, caching keys, and mutations. Keep the fetchers separate from the query hooks for mockability.

### E. Store (`src/store/`)
*   **Purpose**: Global, non-persisted client-side UI states.
*   **Rules**:
    *   Keep stores small and single-purpose (e.g., `useSidebarStore`, `useAuthStore`, `useLocaleStore`). Do not build a single monolithic store containing all unrelated properties.
    *   Avoid placing server cache data in Zustand. Use TanStack Query for server state and Zustand strictly for UI state.

---

## 3. General Scaling Rules

1.  **Strict Path Aliases**: Configure Vite and TypeScript path aliases (e.g. `@/components/*`, `@/features/*`) to avoid long, unmaintainable relative imports (`../../../../components/Button`).
2.  **Explicit Imports/Exports**: Prefer named exports over default exports. Named exports improve autocomplete tooling, IDE refactoring reliability, and make it easier to search the project.
3.  **Strict Dependency Flow**:
    *   `pages` can import from `features`, `components`, `store`, and `api`.
    *   `features` can import from `components`, `store`, and `api`.
    *   `components` must **never** import from `features`, `pages`, or `store`.
    *   `store` and `api` must **never** import from `components`, `features`, or `pages`.
