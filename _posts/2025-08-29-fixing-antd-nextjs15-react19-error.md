---
layout: post
title: "⚡ Fixing Ant Design + Next.js 15 + React 19 Runtime Error"
date: 2025-08-29 14:00:00 +0500
categories: [React, Next.js, BugFix]
tags: [nextjs, react19, antd, bugfix, patch, runtime-error]
---

👋 **Hi again!**  
Today I ran into a painful runtime/build issue while integrating **Ant Design (antd)** into a **Next.js 15.5.2** project running **React 19.1.0**. It looked like a simple import problem at first — but turned out to be a mixture of **Server/Client boundaries**, **React 19 compatibility**, and a CSS/SSRed-first-paint detail. Here's exactly what I saw, what I tried, and how I finally fixed it. 🧩

---

## 💥 The Problem

I opened the app after adding Ant Design and saw this runtime error:

```bash
Element type is invalid: expected a string (for built-in components) or
a class/function (for composite components) but got: undefined.
You likely forgot to export your component from the file it's defined in,
or you might have mixed up default and named imports.

Check the render method of `RootLayout`.
```

At the same time I also encountered a build error in other attempts:

```bash
You are attempting to export "metadata" from a component marked with "use client", which is disallowed.
```

So two symptoms:

- Components like `Layout`, `Layout.Header` were becoming `undefined` at runtime.
- I had a `metadata` vs `"use client"` conflict when I tried to make the layout a client component.

---

## 🧪 What I Tried (That Didn’t Help)

I tried many things before landing on the correct approach:

- Importing `Layout` directly inside `app/layout.tsx` (server) → ❌ runtime undefined error.
- Switching import styles: `import Layout from "antd/es/layout"` vs `import { Layout } from "antd"` — mixed results and confusion.
- Marking `layout.tsx` with `"use client"` so AntD could run there → ❌ Next.js build error because `metadata` is exported in the same file.
- Skipping the React 19 compatibility adjustments → ❌ Modal/Message/wave features or static methods misbehaved.
- Importing only `antd/dist/reset.css` → ✅ fixed some styling quirks but **did not fix the runtime/undefined error**.
- Trying older AntD versions → still incompatible or introduced regressions.

All of these were frustrating but useful to understand the root causes.

---

## 🔍 What Was Actually Wrong

After digging I found the real causes:

1. **Server / Client mismatch (Next.js App Router)**

   - `app/layout.tsx` is a **server component** by default (and needs to export `metadata`). AntD components rely on browser APIs, so they **must** be used in **client components** (`"use client"`). Trying to render AntD in a server component (or marking layout as client while exporting `metadata`) created conflicts.

2. **React 19 compatibility**

   - React 19 changed internal exports/behavior. AntD v5.x needed a compatibility patch to work cleanly with React 19 — otherwise some internals can produce `undefined` component exports or broken static functions.

3. **Imports & dot-syntax**

   - Some dot-syntax usages (`<Layout.Header />`) and inconsistent import paths can behave differently with bundlers and React 19. Using consistent imports and destructuring inside client components is more reliable.

4. **First paint style flash (FOUC)**
   - AntD uses CSS-in-JS / tokens; if server-rendered CSS isn’t extracted or antd reset is not imported early, you may get flash-of-unstyled content. This is separate but related UX problem.

---

## ✅ What Finally Worked (Full step-by-step solution)

Follow these exact steps — this is the **exact working setup** I used.

---

### 📂 Minimal project structure I used

```bash
app/
├─ layout.tsx            # Server component (metadata + wrapper)
├─ page.tsx              # Server page that uses the client layout
└─ components/
├─ AppLayout.tsx     # Client component (AntD Layout)
├─ AppHeader.tsx     # Client
└─ AppFooter.tsx     # Client
styles/
└─ globals.css           # Import antd reset + global styles
package.json
```

---

### 1️⃣ Install packages

```bash
npm install antd @ant-design/v5-patch-for-react-19
```

(If you want the convenience CLI many use: `npx antd patch` — this applies compatibility patches. I used that too. If the CLI isn't available in your environment, installing the patch package and importing it once in a client entry is equivalent.)

Then (optional but commonly used):

```bash
# many devs run this to apply local compatibility adjustments
npx antd patch
```

---

### 2️⃣ `styles/globals.css` — import AntD reset early

```css
/* styles/globals.css */
@import "antd/dist/reset.css";

/* (If you're using Tailwind, include its layers appropriately)
@tailwind base;
@tailwind components;
@tailwind utilities;
*/

html,
body {
  height: 100%;
  margin: 0;
  font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue",
    Arial;
}
```

> Importing `antd/dist/reset.css` early helps avoid a flash-of-unstyled content.

---

### 3️⃣ `app/layout.tsx` — keep this as **server** (no `use client`)

```tsx
// app/layout.tsx (Server)
import type { Metadata } from "next";
import "../styles/globals.css";

export const metadata: Metadata = {
  title: "Antd + Next.js 15 Demo",
  description: "Minimal setup with React 19",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

**Important:** do **not** add `"use client"` to this file — keep it server-only so `metadata` export remains valid.

---

### 4️⃣ `app/components/AppLayout.tsx` — AntD UI must be in a client component

```tsx
// app/components/AppLayout.tsx
"use client";

import { Layout } from "antd";

const { Header, Content, Footer } = Layout;

export default function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <Layout style={{ minHeight: "100vh" }}>
      <Header style={{ color: "#fff", padding: "0 16px" }}>
        <div style={{ color: "#fff", fontWeight: 700 }}>MyApp</div>
      </Header>

      <Content style={{ padding: 20 }}>{children}</Content>

      <Footer style={{ textAlign: "center" }}>
        © {new Date().getFullYear()} MyApp
      </Footer>
    </Layout>
  );
}
```

- `"use client"` allows AntD to access DOM/browser APIs.
- Destructure `Header`, `Content`, `Footer` from `Layout` in the client component to avoid dot-syntax pitfalls.

---

### 5️⃣ Example small header & footer (client components)

**`app/components/AppHeader.tsx`**

```tsx
"use client";

import { Menu } from "antd";
import Link from "next/link";

const items = [
  { key: "home", label: <Link href="/">Home</Link> },
  { key: "about", label: <Link href="/about">About</Link> },
  { key: "contact", label: <Link href="/contact">Contact</Link> },
];

export default function AppHeader() {
  return (
    <div className="flex items-center px-4">
      <div className="text-white font-bold">MyApp</div>
      <Menu
        mode="horizontal"
        theme="dark"
        items={items}
        className="ml-auto"
      />
    </div>
  );
}
```

**`app/components/AppFooter.tsx`**

```tsx
"use client";

export default function AppFooter() {
  return (
    <div style={{ textAlign: "center" }}>
      Built with ❤️ using Ant Design + Next.js
    </div>
  );
}
```

(You can use these inside `AppLayout` if you want separate header/footer components.)

---

### 6️⃣ `app/page.tsx` — server page using client layout

```tsx
// app/page.tsx
import AppLayout from "./components/AppLayout";

export default function Page() {
  return (
    <AppLayout>
      <h1 style={{ fontSize: 28 }}>Hello from Next.js 15 + Antd 🎉</h1>
      <p>Everything is working — no undefined errors, no metadata conflicts.</p>
    </AppLayout>
  );
}
```

---

### 7️⃣ Optional: import the React-19 patch in a client entry (alternate to `npx antd patch`)

If `npx antd patch` was not run, you can import the patch package once in a client entry (for example in `app/providers.tsx`):

```tsx
// app/providers.tsx
"use client";
import "@ant-design/v5-patch-for-react-19";
import { ConfigProvider, theme } from "antd";

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <ConfigProvider
      theme={{
        algorithm: theme.defaultAlgorithm,
      }}
    >
      {children}
    </ConfigProvider>
  );
}
```

Then wrap pages with `<Providers>` (or import Providers in `app/page.tsx`) so the patch is applied on the client.

---

## ⏳ What’s Next / Extra Notes

- **AntdRegistry / SSR styles**: if you see style flash (FOUC), consider using `@ant-design/nextjs-registry` or AntD’s recommended registry approach to collect and inject styles during server render. This prevents style flicker on first paint.
- **Avoid making `layout.tsx` a client component** (don’t add `"use client"` there) because exporting `metadata` from a client component is disallowed by Next.js.
- **If you see `Element type is invalid`**: re-check that AntD components are used inside client components and that you applied the React 19 patch (`npx antd patch` or import package).
- **If `npx antd patch` is not available**: `npm install @ant-design/v5-patch-for-react-19` and import it once in a top-level client file (see `providers.tsx` above).

---

## 🙋‍♀️ Final Note

This happened on:

- 🧩 `Next.js 15.5.2`
- ⚛️ `React 19.1.0`
- 🧭 `Ant Design 5.27.1` (patched for React 19)

After applying the patch and moving AntD into client components, everything worked smoothly — Header, Content and Footer render correctly, no `metadata` build error, and no `undefined` components.

Hope this saves you the time I wasted debugging — and makes your AntD + Next.js 15 setup smooth.
📝 Read this blog on [madiha-m.github.io](https://madiha-m.github.io)

— Madiha 💙
