# React Router Routing Modes

React Router (v7+) offers three primary routing modes: **Browser Router**, **Hash Router**, and **Memory Router**. Each mode manages the navigation state differently and has specific implications for deployment, SEO, and integration.

---

## 1. Browser Router (`BrowserRouter` / `createBrowserRouter`)

Uses the HTML5 History API (`pushState`, `replaceState`, and `popstate` events) to keep the UI in sync with the URL.

### Pros:
*   **Clean URLs**: Paths look standard (e.g., `/settings`, `/documents/articles/123`), without any special character separators.
*   **Standards-Compliant**: Integrates fully with standard web platform utilities (caching, link prefetching, etc.).
*   **SSR Ready**: Required for Server-Side Rendering (SSR) or Static Site Generation (SSG).
*   **Security & Cookies**: Better compatibility with domain-based cookie authentication policies and OAuth redirect URIs that require clean paths.

### Cons:
*   **Server Configuration Required**: If a user reloads `/documents/articles/123` directly, the web server (e.g., Nginx, Apache, or the Rust Axum service) must be configured to fall back and serve the entry `index.html` file. Otherwise, the server will return a `404 Not Found` error because the path does not map to a real file on the disk.

---

## 2. Hash Router (`HashRouter` / `createHashRouter`)

Uses the hash portion of the URL (e.g., `/#/settings`, `/#/documents/articles/123`) to manage navigation.

### Pros:
*   **Zero Server Configuration**: The browser never sends the hash fragment (`#...`) to the server in the HTTP request. Therefore, the web server only needs to serve `index.html` at the root path `/`. Direct reloads on any subpath work out of the box without any special server fallback rules.
*   **Extremely Easy Deployment**: Ideal for simple static hosting (e.g., AWS S3 without CloudFront redirects, GitHub Pages, or direct file serving).

### Cons:
*   **Aesthetics**: The `#` in URLs looks dated and less professional.
*   **No SSR**: Since the server does not receive the hash fragment, it cannot pre-render specific pages based on the requested URL.
*   **SEO Limitations**: Web crawlers generally ignore the hash fragment or treat it as the same page, which limits crawlability (though this is typically not an issue for authenticated admin consoles).
*   **OAuth Redirects**: Some identity providers (IdPs) and OAuth services do not allow hashes in redirect URLs, making SSO authentication harder to set up.

---

## 3. Memory Router (`MemoryRouter` / `createMemoryRouter`)

Manages history entirely in memory. It does not read from or write to the browser's address bar.

### Pros:
*   **Environment Agnostic**: Works in environments without a URL bar (e.g., React Native, Electron, mobile wrappers, VS Code extensions, or Chrome browser extension popups).
*   **Excellent for Testing**: Ideal for running unit and integration tests (e.g., Vitest, Jest) because it allows simulating navigation without touching the global browser history.

### Cons:
*   **No Address Bar Sync**: The URL bar does not update as the user navigates, preventing users from bookmarking a specific view or sharing a direct link.
*   **State Loss on Reload**: If the user reloads the page, the application state is reset, and they are returned to the initial route.

---

## Analysis for Luminair UI (CMS Admin Console)

Luminair UI is a **Schema-Driven CMS Admin Console**. For this type of project, the evaluation points are as follows:

| Criteria | Browser Router | Hash Router | Memory Router |
| :--- | :--- | :--- | :--- |
| **URL Cleanliness** | Excellent (`/documents/articles`) | Poor (`/#/documents/articles`) | N/A (No URL updates) |
| **Deep-Linking** | Excellent | Good | Impossible (State lost on refresh) |
| **Deployment Complexity**| Low-Medium (Needs server fallback) | Low (Zero-config) | Low |
| **SSO / OAuth Integration**| Native & Easy | Tricky (IdPs reject hash) | Tricky (Must sync tokens manually) |
| **Testing** | Hard (needs browser context) | Hard (needs browser context) | Perfect (Zero side-effects) |

### Recommendation for Luminair UI: **Browser Router**

We recommend **Browser Router** for the following reasons:
1. **SSO and JWKS Authentication**: As a premium CMS, Luminair integrates with modern SSO providers (OAuth2/OIDC). Many Identity Providers strictly forbid `#` in their redirect URIs. Using `BrowserRouter` ensures seamless integration.
2. **Deep-linking and Collaboration**: Content creators constantly copy and paste links to specific documents or settings screens to share with other editors. Clean, deep-linkable URLs are critical for productivity.
3. **Integration with Axum Backend**: The backend is built using Rust (`axum`). Serving static SPA files with a fallback handler in Axum is trivial (e.g., using `tower_http::services::ServeDir` with a fallback to `index.html`), meaning the server configuration overhead is virtually zero.
