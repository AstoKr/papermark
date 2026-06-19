# app — app

# Root Layout (`app/layout.tsx`)

The root layout is the foundation of every page in this Next.js application. In the App Router architecture, this component wraps all other pages and UI, making it the single entry point for the document's HTML structure.

## Purpose

This module serves three essential functions:

1. **Document Shell** — Provides the `<html>` and `<body>` wrapper that wraps every page in the application
2. **SEO Metadata** — Configures page title, description, Open Graph tags, and Twitter Card data for search engines and social sharing
3. **Global Styles** — Imports the application's global CSS stylesheet

## Key Components

### Metadata Configuration

The `metadata` export tells browsers, search engines, and social platforms how to display and index the site:

```typescript
export const metadata: Metadata = {
  metadataBase: new URL("https://www.papermark.com"),
  title: "Papermark | The Open Source DocSend Alternative",
  description: data.description,
  openGraph: { /* social media preview data */ },
  twitter: { /* Twitter Card data */ },
};
```

This metadata is inherited by all pages unless they export their own `metadata` object to override specific values.

### Font Loading

```typescript
const inter = Inter({ subsets: ["latin"] });
```

Uses `next/font/google` to load the Inter typeface. This approach is preferred over `<link>` tags because:

- Fonts are automatically self-hosted at build time, eliminating external requests
- Font files are automatically optimized (subsetting, swapping)
- Layout shift is prevented through automatic size adjustment

### Global Styles

```typescript
import "@/styles/globals.css";
```

The `@/` alias points to the `src/` directory. All styles defined in `globals.css` apply to every page in the application.

## Robots.txt

Located at `app/robots.txt`, this file controls which paths search engine crawlers may access:

```
User-Agent: *
Disallow: /register
Disallow: /verify/
Disallow: /auth/
Disallow: /unsubscribe
```

These paths are excluded from indexing:
- `/register` — Registration forms should not be indexed
- `/verify/` — Email verification flows
- `/auth/` — Authentication endpoints
- `/unsubscribe` — Email unsubscribe pages

## Component Signature

```typescript
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
})
```

The `children` prop receives whatever page component matches the current route. Next.js injects the appropriate page content here automatically.

## Relationship to the Rest of the Application

The root layout is referenced by Next.js automatically — there is no manual import anywhere. The framework expects exactly one layout file at the `app/` directory root to wrap all routes.

Every page in the `app/` directory (and its subdirectories) renders inside the `<body>` of this layout. Pages can export their own `metadata` objects to customize their title, description, or other head elements while still inheriting the base configuration.

## Customizing Metadata Per Page

Individual pages can override metadata values:

```typescript
// app/about/page.tsx
export const metadata = {
  title: "About Papermark",
  description: "Learn about our mission",
};

export default function AboutPage() { ... }
```

The page's metadata will be merged with the root layout's metadata, with page-level values taking precedence.