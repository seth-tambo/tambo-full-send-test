# `npx tambo full-send` does not configure `@/` path alias for Vite projects, breaking all component imports

## Problem

Running `npx tambo full-send` on a fresh Vite + React + TypeScript project installs components that use `@/` path alias imports (e.g., `@/components/tambo/message`, `@/lib/utils`), but **does not configure either TypeScript or Vite to resolve the `@/` alias**. Additionally, two installed component files contain import patterns incompatible with the strict TypeScript settings that the `create-vite` `react-ts` template enables by default.

This results in **~59 TypeScript errors** out of the box: 56 from unresolved `@/` imports, 2 from `verbatimModuleSyntax` violations, and 1 from an unused import flagged by `noUnusedLocals`.

## Steps to Reproduce

### 1. Scaffold a fresh Vite project

```bash
bunx create-vite@latest tambo-full-send-test --template react-ts
cd tambo-full-send-test
npm install
```

### 2. Run `tambo full-send`

```bash
npx tambo full-send
```

The CLI will walk through an interactive setup:

```text
🚀 Initializing tambo with full-send mode. Let's get you set up!

Found existing src/ directory

✔ Would you like to use the existing src/ directory for components? Yes
✔ Choose where to connect your app: Cloud (time: 1 minute) — recommended
✔ Already authenticated
✔ Select a project: + Create new project
✔ Project name: tambo-full-send-test
✔ Created project: tambo-full-send-test
✔ API key generated

Detected Vite project

✔ Created new .env.local file
✔ API key saved to .env.local as VITE_TAMBO_API_KEY
✔ Would you like to update/add AGENTS.md and CLAUDE.md for LLMs? Yes
```

### 3. Select components to install

```text
✔ Select the components you want to install:
  message-thread-full, message-thread-panel, message-thread-collapsible, control-bar
```

The CLI installs npm packages, drops 20 component files into `src/components/tambo/`, creates utility files, configures Tailwind CSS v4, and updates `globals.css`:

```text
✔ Installed message-thread-full
✔ Installed message-thread-panel
✔ Installed message-thread-collapsible
✔ Installed control-bar
✔ Installed @tailwindcss/vite
✔ Added @tailwindcss/vite plugin to vite.config.ts
✔ Successfully updated globals.css

✨ Full-send initialization complete!
```

### 4. Observe errors

Open the project in an IDE — all `@/` imports show as unresolved. Or run:

```bash
npx tsc -b
```

**Expected:** Zero type errors — the project should be ready to develop against.
**Actual:** ~59 errors across the installed component files.

## What `tambo full-send` does (and doesn't do)

The CLI does several things correctly during setup:

- Detects the Vite project type
- Installs `@tambo-ai/react`, `@tambo-ai/react-ui-base`, and `@tambo-ai/typescript-sdk`
- Installs `@tailwindcss/vite` and adds the plugin to `vite.config.ts`
- Drops 20 component files into `src/components/tambo/`
- Creates `src/lib/tambo.ts` (component registry) and `src/lib/utils.ts` (cn utility)
- Updates `globals.css` with 84 CSS variables, custom variants, and theme blocks

But it **does not**:

1. Update `tsconfig.app.json` or `vite.config.ts` with the `@/` path alias that all installed components depend on
2. Ensure the installed component code is compatible with the target template's TypeScript strictness settings

## What the installed components expect

Every component file uses `@/` imports to reference siblings and utilities:

```tsx
// src/components/tambo/control-bar.tsx
import { useThreadContentContext } from "@tambo-ai/react-ui-base/thread-content"; // ✅ works
import type { messageVariants } from "@/components/tambo/message";                // ❌ broken
import { MessageInput, ... } from "@/components/tambo/message-input";             // ❌ broken
import { cn } from "@/lib/utils";                                                 // ❌ broken
```

This pattern repeats across all 20 installed files. The `@tambo-ai/*` package imports resolve fine from `node_modules` — only the local `@/` imports are broken.

## Why it's broken

The `react-ts` Vite template ships with a minimal `tsconfig.app.json` that has **no `baseUrl` or `paths`**, and a `vite.config.ts` with **no `resolve.alias`**:

**tsconfig.app.json** (as created by `create-vite`):

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "bundler",
    // ... no baseUrl, no paths
  }
}
```

**vite.config.ts** (after `tambo full-send` modifies it):

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [tailwindcss(), react()],
  // no resolve.alias
})
```

Neither file tells TypeScript or Vite what `@/` means, so every `@/` import fails.

## Additional inconsistency

The CLI's own suggested `App.tsx` code mixes alias and relative imports:

```tsx
// Suggested by tambo full-send:
import { components } from "../../lib/tambo";                              // relative path
import { MessageThreadFull } from "@/components/tambo/message-thread-full"; // @/ alias
```

The relative import for `tambo.ts` would work, but the `@/` imports for the components would not — in the same file.

## Issue 1: Missing `@/` path alias (56 errors)

Every component file uses `@/` imports to reference siblings and utilities:

```tsx
// src/components/tambo/control-bar.tsx
import { useThreadContentContext } from "@tambo-ai/react-ui-base/thread-content"; // ✅ works
import type { messageVariants } from "@/components/tambo/message";                // ❌ broken
import { MessageInput, ... } from "@/components/tambo/message-input";             // ❌ broken
import { cn } from "@/lib/utils";                                                 // ❌ broken
```

This pattern repeats across all 20 installed files. The `@tambo-ai/*` package imports resolve fine from `node_modules` — only the local `@/` imports are broken.

### Why

The `react-ts` Vite template ships with a minimal `tsconfig.app.json` that has **no `baseUrl` or `paths`**, and a `vite.config.ts` with **no `resolve.alias`**:

**tsconfig.app.json** (as created by `create-vite`):

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "bundler"
    // ... no baseUrl, no paths
  }
}
```

**vite.config.ts** (after `tambo full-send` modifies it):

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [tailwindcss(), react()],
  // no resolve.alias
})
```

Neither file tells TypeScript or Vite what `@/` means, so every `@/` import fails to resolve.

### Fix

`npx tambo full-send` should add the `@/` path alias to both config files when it detects a Vite project. The CLI already modifies `vite.config.ts` (to add the Tailwind plugin) and creates new files in `src/`, so it has the access and context to make these additions during the same setup flow.

**1. `tsconfig.app.json`** — add `baseUrl` and `paths`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**2. `vite.config.ts`** — add `resolve.alias`:

```ts
import path from "path";
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [tailwindcss(), react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

## Issue 2: `verbatimModuleSyntax` violation in component template (2 errors)

The `create-vite` `react-ts` template enables `verbatimModuleSyntax: true` in `tsconfig.app.json`. This requires type-only imports to use the `import type` syntax. The installed `message.tsx` component violates this:

```tsx
// src/components/tambo/message.tsx (line 3) — current:
import { ReactTamboThreadMessage, TamboToolUseContent } from "@tambo-ai/react";

// These are only used as type annotations, never as runtime values.
// Should be:
import type { ReactTamboThreadMessage, TamboToolUseContent } from "@tambo-ai/react";
```

### Fix for Issue 2

The component template should use `import type` for type-only imports.

## Issue 3: Unused import flagged by `noUnusedLocals` (1 error)

The `create-vite` `react-ts` template enables `noUnusedLocals: true`. The installed `dictation-button.tsx` imports `React` as a default import but never uses it:

```tsx
// src/components/tambo/dictation-button.tsx (line 3) — current:
import React, { useEffect, useRef } from "react";

// Should be:
import { useEffect, useRef } from "react";
```

### Fix for Issue 3

The component template should not include the unused `React` default import.

## Additional inconsistency in CLI output

The CLI's own suggested `App.tsx` code mixes alias and relative imports:

```tsx
// Suggested by tambo full-send:
import { components } from "../../lib/tambo";                              // relative path
import { MessageThreadFull } from "@/components/tambo/message-thread-full"; // @/ alias
```

The relative import for `tambo.ts` would work, but the `@/` imports for the components would not — in the same suggested snippet.

## Environment

- OS: macOS (Darwin 25.2.0)
- `create-vite` template: `react-ts`

### Installed dependency versions (from `package.json`)

**Runtime dependencies:**

- `react`: ^19.2.4
- `react-dom`: ^19.2.4
- `@tambo-ai/react`: ^1.0.0-rc.5 (resolves to 1.2.3 in node_modules)
- `@tambo-ai/react-ui-base`: ^0.1.0 (resolves to 0.1.6 in node_modules)
- `@tambo-ai/typescript-sdk`: ^0.94.0
- `radix-ui`: ^1.4.3
- `@radix-ui/react-collapsible`: ^1.1.12
- `@radix-ui/react-dropdown-menu`: ^2.1.16
- `@radix-ui/react-popover`: ^1.1.15
- `@radix-ui/react-slot`: ^1.2.4
- `@tiptap/core`: ^3.20.4
- `@tiptap/react`: ^3.20.4
- `class-variance-authority`: ^0.7.1
- `dompurify`: ^3.3.3
- `framer-motion`: ^12.38.0
- `highlight.js`: ^11.11.1
- `lucide-react`: ^0.577.0
- `streamdown`: ^2.5.0
- `use-debounce`: ^10.1.0

**Dev dependencies:**

- `vite`: ^8.0.1
- `typescript`: ~5.9.3
- `@vitejs/plugin-react`: ^6.0.1
- `@tailwindcss/vite`: ^4.2.2
- `tailwindcss`: ^4.2.2
- `clsx`: ^2.1.1
- `tailwind-merge`: ^3.5.0
- `@types/node`: ^24.12.0
- `@types/react`: ^19.2.14
- `@types/react-dom`: ^19.2.3

### Transitive Tambo dependencies (from node_modules)

- `@tambo-ai/client`: 0.1.0 (dependency of `@tambo-ai/react`)
- `@tambo-ai/typescript-sdk`: 0.93.1 (peer of `@tambo-ai/react`, differs from top-level 0.94.0)
- `@modelcontextprotocol/sdk`: 1.27.1 (dependency of `@tambo-ai/react`)
- `@base-ui/react`: ^1.2.0 (dependency of `@tambo-ai/react-ui-base`)
