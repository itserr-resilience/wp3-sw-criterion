# Criterion — Developer Guide

**Document Type**: Technical Reference Documentation  
**Version**: 1.2  
**Last Revision**: March 2026  
**Intended Audience**: Software Engineers, Technical Contributors

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Development Environment Setup](#2-development-environment-setup)
3. [Repository Structure](#3-repository-structure)
4. [Architecture Overview](#4-architecture-overview)
5. [Electron Configuration](#5-electron-configuration)
6. [Preload API Reference](#6-preload-api-reference)
7. [TipTap Editor System](#7-tiptap-editor-system)
8. [State Management](#8-state-management)
9. [Data Persistence](#9-data-persistence)
10. [Testing Framework](#10-testing-framework)
11. [Build and Distribution](#11-build-and-distribution)
12. [Coding Standards](#12-coding-standards)
13. [Debugging Procedures](#13-debugging-procedures)
14. [Contribution Guidelines](#14-contribution-guidelines)
15. [Appendices](#15-appendices)

---

## 1. Product Overview

### Application Description

**Criterion** is a professional desktop application designed for **critical text edition** — the scholarly discipline of reconstructing historical texts from multiple manuscript sources, documenting textual variants, and producing annotated critical editions.

### Primary User Categories

The application is designed to serve the following professional user categories:

- Philologists and textual criticism specialists
- Historians engaged in primary source research
- Academic publishing professionals producing critical editions
- Researchers specializing in classical, medieval, and modern literary studies

### Platform Support Matrix

| Platform | Architecture | Package Format | Status     |
| -------- | ------------ | -------------- | ---------- |
| macOS    | x64, arm64   | `.pkg`         | Production |
| Windows  | x64          | `.exe` (NSIS)  | Production |
| Linux    | x64          | `.deb`         | Production |

---

## 2. Development Environment Setup

### Prerequisites Checklist

Before you begin, you need to install the following tools. Each one plays a specific role in the project:

| #   | Tool                     | Required Version                 | What It Does                                                                                                                                       | Verification         |
| --- | ------------------------ | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| 1   | **Git**                  | 2.30+                            | Source code version control — you'll clone the repository with it                                                                                  | `git --version`      |
| 2   | **nvm**                  | Latest                           | **Node Version Manager** — lets you install and switch between Node.js versions. Critical because this project requires an _exact_ Node.js version | `nvm --version`      |
| 3   | **Node.js**              | **24.11.0** (pinned in `.nvmrc`) | JavaScript runtime powering Electron's main process and all build tooling                                                                          | `node --version`     |
| 4   | **Corepack**             | Built into Node 24+              | A Node.js built-in tool that manages package managers. Used here to enable Yarn 4 Berry                                                            | `corepack --version` |
| 5   | **Yarn**                 | 4.x (Berry)                      | Package manager for installing dependencies. The project uses Yarn 4 "Berry" (not Yarn 1 Classic) with `nodeLinker: node-modules` mode             | `yarn --version`     |
| 6   | **Platform build tools** | —                                | Native compilation tools needed by some Electron dependencies (see [Platform-Specific Setup](#platform-specific-setup) below)                      | —                    |

> **Why nvm instead of installing Node.js directly?**  
> The project pins an exact Node.js version in the `.nvmrc` file (currently `24.11.0`). Using nvm guarantees every developer runs the same version and avoids "works on my machine" issues. When you run `nvm install && nvm use`, nvm reads `.nvmrc` and installs/activates that exact version automatically.

> **Why Yarn 4 Berry?**  
> Yarn 4 Berry is faster and more deterministic than npm or Yarn 1. The project configures it via `.yarnrc.yml` with `nodeLinker: node-modules`, meaning it still creates a traditional `node_modules/` folder (some Electron tooling requires this).

### Platform-Specific Setup

You **must** complete the platform-specific setup for your operating system **before** running the installation steps.

<details>
<summary><strong>macOS</strong></summary>

```bash
# 1. Install Xcode Command Line Tools (required for compiling native modules)
#    This installs gcc, make, and other build essentials.
#    A dialog will appear — click "Install" and wait for it to finish.
xcode-select --install

# 2. Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

# 3. Reload your shell so the `nvm` command becomes available
source ~/.zshrc    # If you use zsh (default on modern macOS)
# source ~/.bashrc # If you use bash instead
```

**Verify it worked:**

```bash
nvm --version   # Should print a version number like 0.40.0
```

</details>

<details>
<summary><strong>Ubuntu/Linux</strong></summary>

```bash
# 1. Install system dependencies (required by Electron and native modules)
#    These libraries are needed for Electron's Chromium runtime to function correctly.
sudo apt update
sudo apt install -y curl build-essential \
  libgtk-3-dev libnotify-dev libnss3 libxss1 libasound2 \
  libatk-bridge2.0-0 libgdk-pixbuf2.0-0 libx11-xcb1 \
  libxcomposite1 libxdamage1 libxrandr2 libxcursor1 \
  libxext6 libxcb1 libnspr4 libdrm2 libcups2 libexpat1

# 2. Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

# 3. Reload your shell
source ~/.bashrc
```

**Verify it worked:**

```bash
nvm --version   # Should print a version number like 0.40.0
```

> **Linux Sandbox Note**: If you encounter sandbox errors when running the app, use `yarn dev-linux` instead of `yarn dev`. This sets `ELECTRON_DISABLE_SANDBOX=1` to bypass Chrome sandbox restrictions on some Linux configurations.

</details>

<details>
<summary><strong>Windows</strong></summary>

1. **Install nvm-windows**: Download the installer from [nvm-windows releases](https://github.com/coreybutler/nvm-windows/releases) and run it
2. **Install Visual Studio Build Tools**: Download from [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/), select **"Desktop development with C++"** workload (this provides the C++ compiler needed by some native Node.js modules)
3. **Open PowerShell as Administrator** and run:

```powershell
nvm install 24.11.0
nvm use 24.11.0
corepack enable
```

**Verify it worked:**

```powershell
nvm version      # Should print 24.11.0
node --version   # Should print v24.11.0
```

</details>

### Step-by-Step First Run

Once your platform-specific setup is complete, follow these steps **in order**:

```bash
# ─── Step 1: Clone the repository ───────────────────────────────────────────
git clone <repository-url> criterion && cd criterion
# Expected: "Cloning into 'criterion'..." — you are now inside the project folder

# ─── Step 2: Install and activate the correct Node.js version ────────────────
nvm install    # Reads .nvmrc → installs Node.js 24.11.0 if not already installed
nvm use        # Activates Node.js 24.11.0 for this terminal session
# Expected: "Now using node v24.11.0 (npm v...)"

# ─── Step 3: Enable Corepack (activates Yarn 4) ─────────────────────────────
corepack enable
# Expected: no output (silence means success)

# ─── Step 4: Install all project dependencies ───────────────────────────────
yarn install
# Expected: Downloads ~200 packages into node_modules/. First run takes 2-5 min.
# You may see "➤ YN0000: · Done ..." when finished.

# ─── Step 5: Launch the development server ──────────────────────────────────
yarn dev       # (Linux: use `yarn dev-linux` if you get sandbox errors)
# Expected: Vite compiles main + preload + renderer processes,
# then Electron opens the Criterion application window.
```

> **Tip**: After `yarn install`, Electron's native dependencies are automatically rebuilt via the `postinstall` script (`electron-builder install-app-deps`). You don't need to do anything extra.

### Understanding the Configuration Files

Here's what the key configuration files in the project root do — you'll encounter them often:

| File                      | Purpose                                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.nvmrc`                  | Pins the exact Node.js version (`24.11.0`). Read by `nvm install` and `nvm use`.                                                                                       |
| `.yarnrc.yml`             | Yarn 4 Berry config. Sets `nodeLinker: node-modules` so dependencies go into a classic `node_modules/` folder.                                                         |
| `.npmrc`                  | npm registry configuration (e.g., custom registry URL).                                                                                                                |
| `package.json`            | Project metadata, all npm scripts, dependencies, and version number.                                                                                                   |
| `yarn.lock`               | Lockfile ensuring deterministic installs — **never edit manually**.                                                                                                    |
| `electron.vite.config.ts` | Build configuration for [electron-vite](https://electron-vite.org/). Defines separate configs for `main`, `preload`, and `renderer` processes, including path aliases. |
| `electron-builder.json`   | Distribution/packaging configuration for [electron-builder](https://www.electron.build/). Defines how installers are created per platform.                             |
| `tsconfig.json`           | Root TypeScript configuration.                                                                                                                                         |
| `tsconfig.node.json`      | TypeScript config for main + preload processes (Node.js target).                                                                                                       |
| `tsconfig.web.json`       | TypeScript config for the renderer process (browser target).                                                                                                           |
| `.env.development`        | Environment variables for **development** builds (`VITE_DEBUG_MODE=true`, etc.).                                                                                       |
| `.env.staging`            | Environment variables for **staging** builds.                                                                                                                          |
| `.env.production`         | Environment variables for **production** builds (`VITE_DEBUG_MODE=false`, etc.).                                                                                       |
| `eslint.config.mjs`       | ESLint 9 flat config for linting rules.                                                                                                                                |
| `.stylelintrc.json`       | Stylelint config for CSS/SCSS linting.                                                                                                                                 |
| `tailwind.config.cjs`     | TailwindCSS configuration with custom color palette.                                                                                                                   |
| `postcss.config.cjs`      | PostCSS config (used by TailwindCSS).                                                                                                                                  |
| `jest.config.js`          | Jest test runner configuration.                                                                                                                                        |
| `.gitlab-ci.yml`          | CI/CD pipeline definition (lint → build → distribute).                                                                                                                 |
| `components.json`         | shadcn/ui component configuration.                                                                                                                                     |

### Environment Variables Quick Reference

The project uses three environment files — each loaded automatically based on the build mode:

| Variable                      | `.env.development` | `.env.staging` | `.env.production` |
| ----------------------------- | ------------------ | -------------- | ----------------- |
| `VITE_APP_ENV`                | `development`      | `staging`      | `production`      |
| `VITE_DEBUG_MODE`             | `true`             | `true`         | `false`           |
| `VITE_LOG_LEVEL`              | `debug`            | `info`         | `error`           |
| `VITE_ENABLE_DEV_TOOLS`       | `true`             | `true`         | `false`           |
| `VITE_ENABLE_MOCK_DATA`       | `false`            | `false`        | `false`           |
| `MAIN_VITE_DOCUMENT_SIGN_KEY` | _(set)_            | _(set)_        | _(set)_           |

> **How are env files selected?** The `cross-env NODE_ENV=<env>` in each script tells electron-vite which `.env.*` file to load. For example, `yarn dev` loads `.env.development`, while `yarn build` loads `.env.production`.

### Troubleshooting Common Installation Issues

| Symptom                        | Cause                               | Fix                                        |
| ------------------------------ | ----------------------------------- | ------------------------------------------ |
| `EACCES` permission errors     | npm global installation permissions | Use nvm — **never** use `sudo npm install` |
| Sandbox errors (Linux)         | Chrome sandbox restrictions         | Use `yarn dev-linux` instead of `yarn dev` |
| Native module errors           | Node.js version mismatch            | Run `nvm use && yarn install` to realign   |
| `corepack` not found           | Old Node.js version (< 16.10)       | Update to Node 24+ via `nvm install`       |
| Build hangs or corrupted state | Corrupted cache or build output     | Run `yarn clean && yarn install`           |
| `yarn` command not found       | Corepack not enabled                | Run `corepack enable` then retry           |
| `EPERM` errors (Windows)       | Terminal not running as Admin       | Run PowerShell as Administrator            |

---

## 3. Repository Structure

```
criterion/
├── src/
│   ├── main/                    # Electron Main Process (Node.js)
│   │   ├── index.ts             # Entry point (~3000 lines) — IPC registry
│   │   ├── document/            # Document operations, TEI export
│   │   ├── menu/                # Application menu definitions
│   │   │   └── items/           # Menu item factories
│   │   ├── workers/             # Background workers (PDF, etc.)
│   │   └── shared/              # Cross-process utilities
│   │
│   ├── preload/                 # Security Bridge
│   │   ├── index.ts             # contextBridge API definitions
│   │   └── index.d.ts           # TypeScript declarations for window.*
│   │
│   └── renderer/src/            # React UI (Chromium)
│       ├── pages/               # Route components
│       │   └── editor/          # Main editor page
│       │       ├── store/       # Editor-specific Redux slices
│       │       └── provider/    # Editor React Context
│       ├── components/          # Reusable components
│       │   └── ui/              # shadcn/ui components
│       ├── lib/                 # Libraries
│       │   ├── editor/          # TipTap editor instances
│       │   │   ├── extensions/  # Custom editor extensions
│       │   │   ├── components/  # Editor UI components
│       │   │   ├── controllers/ # Editor controllers
│       │   │   ├── hooks/       # Editor-specific hooks
│       │   │   └── shared/      # Shared editor utilities
│       │   └── tiptap/          # Generic TipTap extensions
│       │       ├── extensions/  # Node extensions
│       │       └── marks/       # Mark extensions
│       ├── store/               # Global Redux store
│       ├── hooks/               # Custom React hooks
│       └── views/               # Secondary windows (About, Preferences, etc.)
│
├── i18n/                        # Translations (de, en, es, fr, it)
├── resources/                   # JSON configs (styles, metadata, etc.)
├── buildResources/              # Build assets
│   ├── printPreview/            # PDF generation (Java, TinyTeX)
│   ├── templates/               # Document templates
│   └── styles/                  # Default styles
└── docs/                        # Documentation
```

### Strict Boundaries

| Layer       | Can Import From                         | Cannot Import From            |
| ----------- | --------------------------------------- | ----------------------------- |
| `main/`     | Node.js, Electron main APIs             | `renderer/`, `preload/`       |
| `preload/`  | Electron `contextBridge`, `ipcRenderer` | `main/`, `renderer/`, Node.js |
| `renderer/` | React, window.\* APIs                   | `main/`, `preload/`, Node.js  |

**Security Notice**: Violations of these boundaries constitute security vulnerabilities. The renderer process must never have direct access to Node.js APIs.

---

## 4. Architecture Overview

### Three-Process Model

```
┌─────────────────────────────────────────────────────────────────┐
│                         MAIN PROCESS                             │
│  (Node.js + Electron APIs)                                       │
│  • File system operations                                        │
│  • IPC message handling                                          │
│  • Window management                                             │
│  • Native menus                                                  │
│  • Background workers                                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │ IPC (invoke/handle)
┌──────────────────────────┴──────────────────────────────────────┐
│                       PRELOAD SCRIPT                             │
│  (Isolated context with contextBridge)                           │
│  • Exposes safe APIs to renderer                                 │
│  • Type-safe interface definitions                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │ window.* APIs
┌──────────────────────────┴──────────────────────────────────────┐
│                      RENDERER PROCESS                            │
│  (Chromium + React)                                              │
│  • UI components                                                 │
│  • TipTap editors                                                │
│  • Redux state                                                   │
│  • No Node.js access                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Example: Document Persistence

```
User clicks Save
       │
       ▼
┌──────────────────┐
│ React Component  │  calls window.doc.saveDocument(data)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Preload Bridge   │  ipcRenderer.invoke('document:save', data)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Main Process     │  ipcMain.handle('document:save', handler)
│ document-manager │  → validates → writes to disk → returns path
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ React Component  │  receives result, updates UI
└──────────────────┘
```

---

## 5. Electron Configuration

### Security Configuration

| Setting            | Value   | Rationale                         |
| ------------------ | ------- | --------------------------------- |
| `contextIsolation` | `true`  | Preload runs in isolated context  |
| `nodeIntegration`  | `false` | Renderer has no Node.js access    |
| `sandbox`          | `false` | Required for some native features |
| `webSecurity`      | `true`  | Enforces same-origin policy       |

### Window Management

Each document opens in a separate `WebContentsView`:

```typescript
// Main process creates views
const view = new WebContentsView(webPreferences);
mainWindow.contentView.addChildView(view);

// Tab state persisted via electron-store
store.set("tabs", tabsArray);
```

### IPC Channel Naming Convention

```
namespace:action[:subaction]

Examples:
  document:save
  document:export:tei
  tabs:new
  menu:update:apparatuses
```

---

## 6. Preload API Reference

The preload script exposes **11 namespaces** to the renderer via `window.*`:

### Namespace Overview

| Namespace                  | Purpose                           | Method Count |
| -------------------------- | --------------------------------- | ------------ |
| `window.tabs`              | Tab lifecycle management          | 6            |
| `window.menu`              | Menu state updates                | 19           |
| `window.system`            | System utilities (fonts, dialogs) | 8            |
| `window.application`       | App state (toolbar, zoom)         | 9            |
| `window.doc`               | Document operations               | 75+          |
| `window.theme`             | Theme management                  | 2            |
| `window.preferences`       | User preferences                  | 4            |
| `window.tooltip`           | Tooltip window control            | 3            |
| `window.keyboardShortcuts` | Shortcut management               | 4            |
| `window.debug`             | Debug utilities                   | 4            |

### Key APIs

#### `window.doc` — Document Operations

```typescript
// Save/Load
window.doc.save(): Promise<SaveResult>
window.doc.saveAs(): Promise<SaveResult>
window.doc.saveDocument(data: DocumentData): Promise<void>
window.doc.loadDocument(data: DocumentData): Promise<void>

// Content
window.doc.getContent(): Promise<JSONContent>
window.doc.setContent(content: JSONContent): Promise<void>

// Apparatuses
window.doc.getApparatusesTypes(): Promise<ApparatusType[]>
window.doc.getApparatusNotes(apparatusId: string): Promise<Note[]>
window.doc.setApparatusNotes(apparatusId: string, notes: Note[]): Promise<void>
window.doc.getApparatusContent(apparatusId: string): Promise<JSONContent>
window.doc.setApparatusContent(apparatusId: string, content: JSONContent): Promise<void>

// Export
window.doc.exportToTei(): Promise<string>
window.doc.printPdf(): Promise<void>
window.doc.savePdf(): Promise<void>  // Shows save dialog

// Styles & Templates
window.doc.getDocumentStyles(): Promise<Style[]>
window.doc.setDocumentStyles(styles: Style[]): Promise<void>
window.doc.importStyles(): Promise<void>  // Shows file dialog
window.doc.exportStyles(): Promise<void>
window.doc.getTemplate(): Promise<Template>

// Sigla
window.doc.getSigla(): Promise<Siglum[]>
window.doc.setSigla(sigla: Siglum[]): Promise<void>
window.doc.importSigla(): Promise<void>
window.doc.exportSigla(): Promise<void>

// Find & Replace
window.doc.find(query: string, options?: FindOptions): Promise<FindResult[]>
window.doc.replace(query: string, replacement: string): Promise<void>
window.doc.replaceAll(query: string, replacement: string): Promise<number>
```

#### `window.system` — System Utilities

```typescript
window.system.getInstalledFonts(): Promise<string[]>
window.system.openFileDialog(options?: DialogOptions): Promise<string[]>
window.system.saveFileDialog(options?: DialogOptions): Promise<string>
window.system.log(level: string, ...args: unknown[]): void
window.system.openExternal(url: string): Promise<void>
window.system.getAppVersion(): Promise<string>
window.system.getPlatform(): string
window.system.getLocale(): string
```

#### `window.tabs` — Tab Management

```typescript
window.tabs.newTab(options?: TabOptions): Promise<string>
window.tabs.closeTab(tabId: string): Promise<void>
window.tabs.selectTab(tabId: string): Promise<void>
window.tabs.reorderTabs(tabIds: string[]): Promise<void>
window.tabs.getSelectedTabId(): Promise<string>
window.tabs.getAllContentViewsIds(): Promise<string[]>
```

#### `window.debug` — Debug Utilities

```typescript
window.debug.getLayoutTabs(): Promise<TabLayout[]>
window.debug.getCurrentTabs(): Promise<Tab[]>
window.debug.testTabRestoration(): Promise<void>
window.debug.forceSaveTabs(): Promise<void>
```

### Adding a New IPC Channel

1. **Main process** (`src/main/index.ts`):

```typescript
ipcMain.handle("namespace:action", async (event, arg1, arg2) => {
  // Implementation
  return result;
});
```

2. **Preload** (`src/preload/index.ts`):

```typescript
const namespaceApi = {
  action: (arg1: Type1, arg2: Type2): Promise<ResultType> =>
    ipcRenderer.invoke("namespace:action", arg1, arg2),
};
contextBridge.exposeInMainWorld("namespace", namespaceApi);
```

3. **Type declaration** (`src/preload/index.d.ts`):

```typescript
interface NamespaceApi {
  action(arg1: Type1, arg2: Type2): Promise<ResultType>;
}
declare global {
  interface Window {
    namespace: NamespaceApi;
  }
}
```

---

## 7. TipTap Editor System

### Editor Instances

Criterion uses multiple TipTap editor instances for different contexts:

| Editor      | File                                     | Purpose                   |
| ----------- | ---------------------------------------- | ------------------------- |
| Main Text   | `lib/editor/main-text-editor.tsx`        | Primary document content  |
| Apparatus   | `lib/editor/apparatus-text-editor.tsx`   | Critical apparatus notes  |
| Manuscript  | `lib/editor/manuscript-text-editor.tsx`  | Manuscript transcriptions |
| Siglum      | `lib/editor/siglum-text-editor.tsx`      | Manuscript identifiers    |
| Description | `lib/editor/description-text-editor.tsx` | Metadata descriptions     |

### Custom Extensions Inventory

#### Node Extensions (`lib/editor/extensions/`)

| Extension                 | Type | Purpose                     |
| ------------------------- | ---- | --------------------------- |
| `apparatus-entry.tsx`     | Node | Apparatus entry container   |
| `apparatus-paragraph.tsx` | Node | Apparatus paragraph         |
| `lemma.tsx`               | Node | Lemma (base text reference) |
| `siglum.tsx`              | Node | Manuscript siglum           |
| `page-break.tsx`          | Node | Manual page breaks          |
| `reading-separator.tsx`   | Node | Reading variant separator   |
| `reading-type.tsx`        | Node | Reading type indicator      |
| `text-note.tsx`           | Node | Inline text notes           |
| `toc-paragraph.tsx`       | Node | Table of contents entries   |

#### Mark Extensions (`lib/editor/extensions/`)

| Extension               | Type | Purpose             |
| ----------------------- | ---- | ------------------- |
| `bookmark-mark.ts`      | Mark | Document bookmarks  |
| `comment-mark.tsx`      | Mark | Inline comments     |
| `search.tsx`            | Mark | Search highlighting |
| `custom-text-align.tsx` | Mark | Text alignment      |

#### Generic TipTap Extensions (`lib/tiptap/`)

| Extension                        | Purpose                        |
| -------------------------------- | ------------------------------ |
| `character-spacing-extension.ts` | Character spacing control      |
| `line-spacing-extension.ts`      | Line height control            |
| `line-number-extension.ts`       | Line numbering                 |
| `heading-extension.ts`           | Custom headings                |
| `indent-extension.ts`            | Paragraph indentation          |
| `custom-subscript.ts`            | Subscript formatting           |
| `custom-superscript.ts`          | Superscript formatting         |
| `extended-bullet-list.ts`        | Enhanced bullet lists          |
| `extended-ordered-list.ts`       | Enhanced numbered lists        |
| `extended-list-item.ts`          | Enhanced list items            |
| `ligature-mark.ts`               | Ligature support               |
| `section-divider.tsx`            | Section dividers               |
| `non-printable-character.ts`     | Special character display      |
| `editor-transformation.ts`       | Editor content transformations |
| `list-marker-sync-extension.ts`  | List marker synchronization    |
| `text-style-extended.ts`         | Extended text style attributes |

#### TipTap Node Extensions (`lib/tiptap/extensions/`)

| Extension                | Purpose                   |
| ------------------------ | ------------------------- |
| `paragraph-extension.ts` | Custom paragraph handling |

#### TipTap Mark Extensions (`lib/tiptap/marks/`)

| Extension                | Purpose                |
| ------------------------ | ---------------------- |
| `citation-mark.ts`       | Citation formatting    |
| `custom-style-mark.ts`   | Custom inline styles   |
| `letter-spacing-mark.ts` | Letter spacing control |
| `link-mark.ts`           | Hyperlink support      |

### Creating a New Extension

```typescript
// src/renderer/src/lib/editor/extensions/my-extension.tsx
import { Node, mergeAttributes } from "@tiptap/core";

export const MyExtension = Node.create({
  name: "myExtension",
  group: "block",
  content: "inline*",

  addAttributes() {
    return {
      myAttribute: { default: null },
    };
  },

  parseHTML() {
    return [{ tag: "my-extension" }];
  },

  renderHTML({ HTMLAttributes }) {
    return ["my-extension", mergeAttributes(HTMLAttributes), 0];
  },

  addCommands() {
    return {
      setMyExtension:
        (attrs) =>
        ({ commands }) => {
          return commands.insertContent({
            type: this.name,
            attrs,
          });
        },
    };
  },
});
```

### Content Storage

Editor content is stored as TipTap `JSONContent`:

```typescript
// NEVER store HTML, always JSON
const content: JSONContent = editor.getJSON();

// Store in Redux
dispatch(setMainTextContent(content));

// Persist via IPC
await window.doc.setMainTextContent(content);
```

---

## 8. State Management

### Dual State Architecture

Criterion uses **two complementary state systems**:

```
┌─────────────────────────────────────────────────────────────┐
│                    REDUX TOOLKIT + SAGA                      │
│  (Global application state)                                  │
│                                                              │
│  src/renderer/src/store/                                     │
│  ├── store.ts          # Store configuration                 │
│  ├── rootReducers.ts   # Combined reducers                   │
│  └── rootSaga.ts       # Root saga                           │
│                                                              │
│  src/renderer/src/pages/editor/store/                        │
│  ├── editor/editor.slice.ts        # Document state          │
│  ├── comment/comments.slice.ts     # Comments                │
│  ├── bookmark/bookmark.slice.ts    # Bookmarks               │
│  └── pagination/pagination.slice.ts # Pagination             │
└─────────────────────────────────────────────────────────────┘
                           +
┌─────────────────────────────────────────────────────────────┐
│                    REACT CONTEXT + useReducer                │
│  (Editor-local UI state)                                     │
│                                                              │
│  src/renderer/src/pages/editor/provider/                     │
│  ├── context.ts        # Context definition                  │
│  ├── state.ts          # Initial state                       │
│  ├── reducer.ts        # State reducer                       │
│  └── actions/          # Action creators                     │
│                                                              │
│  Usage: const [state, dispatch] = useEditor();               │
└─────────────────────────────────────────────────────────────┘
```

### When to Use What

| Use Redux For          | Use Context For                    |
| ---------------------- | ---------------------------------- |
| Document content       | Editor UI state (toolbars, panels) |
| Comments & bookmarks   | Temporary selections               |
| User preferences       | Modal/dialog state                 |
| Data persisted to disk | Ephemeral UI state                 |

### Redux Slice Pattern

```typescript
// src/renderer/src/pages/editor/store/editor/editor.slice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface EditorState {
  mainTextContent: JSONContent | null;
  isDirty: boolean;
  lastSaved: string | null;
}

const initialState: EditorState = {
  mainTextContent: null,
  isDirty: false,
  lastSaved: null,
};

export const editorSlice = createSlice({
  name: "editor",
  initialState,
  reducers: {
    setMainTextContent: (state, action: PayloadAction<JSONContent>) => {
      state.mainTextContent = action.payload;
      state.isDirty = true;
    },
    markSaved: (state) => {
      state.isDirty = false;
      state.lastSaved = new Date().toISOString();
    },
  },
});
```

---

## 9. Data Persistence

### Document Format Specification: `.critx`

Criterion documents use the `.critx` extension with JSON structure:

```typescript
interface CritxDocument {
  // Header
  signature: "CRITX";
  documentVersion: "1.0";
  appVersion: string;

  // Content
  mainText: JSONContent;
  apparatuses: Apparatus[];
  annotations: {
    comments: Comment[];
    bookmarks: Bookmark[];
  };

  // Configuration
  metadata: DocumentMetadata;
  template: TemplateConfig;
  styles: StyleDefinition[];
  pageSetup: PageSetup;

  // References
  sigla: Siglum[];
  bibliography: BibEntry[];
}
```

### App Settings Persistence

Application settings are stored via `electron-store`:

```typescript
// src/main/store.ts
import Store from "electron-store";

const store = new Store({
  name: "criterion-settings",
  defaults: {
    theme: "system",
    recentFiles: [],
    windowBounds: { width: 1200, height: 800 },
    tabs: [],
  },
});
```

### File Locations

| Data     | macOS                                      | Windows                           | Linux                            |
| -------- | ------------------------------------------ | --------------------------------- | -------------------------------- |
| Settings | `~/Library/Application Support/Criterion/` | `%APPDATA%/Criterion/`            | `~/.config/Criterion/`           |
| Logs     | `~/Library/Logs/Criterion/`                | `%APPDATA%/Criterion/logs/`       | `~/.local/share/Criterion/logs/` |
| Cache    | `~/Library/Caches/Criterion/`              | `%LOCALAPPDATA%/Criterion/Cache/` | `~/.cache/Criterion/`            |

---

## 10. Testing Framework

### Testing Stack Overview

| Tool                  | Purpose                   |
| --------------------- | ------------------------- |
| Jest 30.x             | Test runner               |
| React Testing Library | Component testing         |
| `ts-jest`             | TypeScript transformation |

### Running Tests

```bash
# All tests
yarn test

# Watch mode
yarn test --watch

# Coverage report
yarn test --coverage

# Single file
yarn test path/to/file.test.ts
```

### Test File Locations

Tests are **co-located** with source files:

```
src/renderer/src/lib/tiptap/
├── character-spacing-extension.ts
├── character-spacing-extension.test.ts    # ← Test file
├── line-spacing-extension.ts
└── line-spacing-extension.test.ts         # ← Test file
```

### Test Pattern

```typescript
// component.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { MyComponent } from './my-component';

describe('MyComponent', () => {
  it('should render with default props', () => {
    render(<MyComponent />);
    expect(screen.getByRole('button')).toBeInTheDocument();
  });

  it('should handle click events', async () => {
    const onClick = jest.fn();
    render(<MyComponent onClick={onClick} />);

    fireEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledTimes(1);
  });
});
```

### Testing TipTap Extensions

```typescript
// extension.test.ts
import { Editor } from "@tiptap/core";
import { Document } from "@tiptap/extension-document";
import { Paragraph } from "@tiptap/extension-paragraph";
import { Text } from "@tiptap/extension-text";
import { MyExtension } from "./my-extension";

describe("MyExtension", () => {
  let editor: Editor;

  beforeEach(() => {
    editor = new Editor({
      extensions: [Document, Paragraph, Text, MyExtension],
    });
  });

  afterEach(() => {
    editor.destroy();
  });

  it("should insert node correctly", () => {
    editor.commands.setMyExtension({ attr: "value" });
    expect(editor.getJSON()).toMatchSnapshot();
  });
});
```

---

## 11. Build and Distribution

### How Builds Work — The Big Picture

Building Criterion is a multi-step pipeline. Understanding the flow helps you debug issues:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  1. TYPECHECK                                                           │
│     yarn typecheck:node  →  TypeScript checks main + preload            │
│     yarn typecheck:web   →  TypeScript checks renderer                  │
├──────────────────────────────────────────────────────────────────────────┤
│  2. VITE BUILD                                                          │
│     electron-vite build  →  Compiles TS to JS for all 3 processes       │
│     Output: out/main/index.js, out/preload/preload.mjs,                 │
│             out/renderer/index.html + assets                            │
├──────────────────────────────────────────────────────────────────────────┤
│  3. ELECTRON-BUILDER                                                    │
│     electron-builder --<platform>  →  Packages the compiled app +       │
│     node_modules + extraResources into a platform installer             │
│     Output: dist/*.exe | dist/*.pkg | dist/*.deb                        │
├──────────────────────────────────────────────────────────────────────────┤
│  4. PACKAGE-BUILD (optional)                                            │
│     node scripts/package-build.cjs <platform> <env>  →  Zips the        │
│     installer artifact into a versioned .zip for distribution           │
│     Output: dist/<platform>-build-<env>-<version>.zip                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Build Commands Reference

These commands **compile TypeScript → JavaScript** but do NOT create installers:

| Command              | Environment | What It Does                                           | Output        |
| -------------------- | ----------- | ------------------------------------------------------ | ------------- |
| `yarn build`         | Production  | Runs typecheck + Vite build with `.env.production`     | `out/` folder |
| `yarn build:dev`     | Development | Same, but with `.env.development` (sourcemaps enabled) | `out/` folder |
| `yarn build:staging` | Staging     | Same, but with `.env.staging`                          | `out/` folder |

> **Note**: The `build` command runs `typecheck` first. If type-checking fails, the build is aborted. Fix all TypeScript errors before building.

### Distribution Prerequisites

Before you can create platform installers, make sure you have the following:

#### 1. Build Resources (`buildResources/`)

The `buildResources/` directory contains essential assets that are bundled into every installer. Here's what's inside and why:

```
buildResources/
├── appIcons/               # Application icons (icon.ico, icon.icns)
├── fileIcons/              # File type icons for .critx files (per-platform)
├── icons/                  # UI icons used by the application
├── styles/                 # Default document styles
├── templates/              # Default document templates
├── entitlements.mac.plist  # macOS security entitlements (camera, microphone, etc.)
├── links.json              # External links configuration
└── printPreview/           # ⬇ PDF generation subsystem (see below)
```

#### 2. Print Preview Subsystem (`buildResources/printPreview/`)

The PDF generation feature requires a **Java runtime** and **TinyTeX** (a minimal LaTeX distribution). These are bundled per-platform:

```
buildResources/printPreview/
├── printpreview-0.0.1-SNAPSHOT.jar   # Java application that generates PDFs
├── jre/                               # Java Runtime Environments
│   ├── linux/                         # JRE for Linux
│   ├── macos/                         # JRE for macOS
│   └── win/                           # JRE for Windows
├── TinyTeX-lin/                       # TinyTeX for Linux
├── TinyTeX-mac/                       # TinyTeX for macOS
├── TinyTeX-win/                       # TinyTeX for Windows
├── fonts/                             # Fonts used in PDF generation
└── pdfjs/                             # PDF.js viewer for print preview
```

> [!IMPORTANT]
> **To obtain a new version of `printpreview-0.0.1-SNAPSHOT.jar`, please refer to its own Developer Guide.** The JAR is a separate Java project maintained independently. If you need to update the PDF generation engine, contact the Print Preview team and follow their build/release process.

> **How does electron-builder know what to include?** The `extraResources` section in `electron-builder.json` specifies which files from `buildResources/` are bundled. Each platform configuration (mac/win/linux) has its own `extraResources` block that includes only the JRE and TinyTeX for that operating system, keeping the installer size down.

#### 3. Code Signing Certificates (for production releases)

Code signing is **optional for development** but **required for production** releases:

**macOS** — Requires an Apple Developer certificate. See [MACOS_CERTIFICATE_GUIDE.md](MACOS_CERTIFICATE_GUIDE.md).

```bash
# Set these environment variables before running dist-mac
export CSC_LINK="path/to/certificate.p12"
export CSC_KEY_PASSWORD="certificate-password"
```

**Windows** — Requires an Authenticode certificate. See [WINDOWS_CERTIFICATE_GUIDE.md](WINDOWS_CERTIFICATE_GUIDE.md).

> **Building without certificates?** You can still create unsigned builds for testing. macOS will show a warning when opening unsigned apps. Windows SmartScreen may block unsigned `.exe` files.

### Distribution Commands — Full Reference

| Command                   | Platform        | Env         | Installer Output | Zip Output                    |
| ------------------------- | --------------- | ----------- | ---------------- | ----------------------------- |
| `yarn dist-win`           | Windows x64     | Production  | `.exe` (NSIS)    | `windows-build-pre-<ver>.zip` |
| `yarn dist-win:dev`       | Windows x64     | Development | `.exe` (NSIS)    | `windows-build-dev-<ver>.zip` |
| `yarn dist-win:staging`   | Windows x64     | Staging     | `.exe` (NSIS)    | `windows-build-stg-<ver>.zip` |
| `yarn dist-linux`         | Linux x64       | Production  | `.deb`           | `linux-build-pre-<ver>.zip`   |
| `yarn dist-linux:dev`     | Linux x64       | Development | `.deb`           | `linux-build-dev-<ver>.zip`   |
| `yarn dist-linux:staging` | Linux x64       | Staging     | `.deb`           | `linux-build-stg-<ver>.zip`   |
| `yarn dist-mac`           | macOS x64+arm64 | Production  | `.pkg` (×2)      | `macos-build-pre-<ver>.zip`   |
| `yarn dist-mac:dev`       | macOS x64+arm64 | Development | `.pkg` (×2)      | `macos-build-dev-<ver>.zip`   |
| `yarn dist-mac:staging`   | macOS x64+arm64 | Staging     | `.pkg` (×2)      | `macos-build-stg-<ver>.zip`   |

### Step-by-Step Distribution Per Platform

#### Windows Distribution

```bash
yarn dist-win
```

What this command does internally:

1. **`cross-env NODE_ENV=production`** — Sets environment to production
2. **`npm run build`** — Runs typecheck + Vite build (creates `out/` folder)
3. **`electron-builder --win --x64`** — Packages everything into an NSIS installer:
   - Bundles compiled code from `out/`
   - Bundles `node_modules` (production only)
   - Copies `extraResources` (i18n, buildResources, **Windows JRE**, **TinyTeX-win**)
   - Creates `dist/Criterion-<version>.exe`
4. **`node scripts/package-build.cjs windows pre`** — Zips the `.exe` into `dist/windows-build-pre-<version>.zip`

**Output**: `dist/Criterion-<version>.exe` + `dist/windows-build-pre-<version>.zip`

#### Linux Distribution

```bash
yarn dist-linux
```

What this command does internally:

1. **`cross-env NODE_ENV=production`** — Sets environment to production
2. **`npm run build`** — Runs typecheck + Vite build
3. **`electron-builder --linux --x64 --config.linux.target=deb`** — Creates a `.deb` package:
   - Includes `postinstall.sh` script (runs after installation on target machine)
   - Bundles **Linux JRE** and **TinyTeX-lin**
   - Declares system dependencies (libgtk-3-0, libnss3, etc.) in the `.deb` metadata
   - Creates `dist/Criterion-<version>.deb`
4. **`node scripts/package-build.cjs linux pre`** — Zips the `.deb`

**Output**: `dist/Criterion-<version>.deb` + `dist/linux-build-pre-<version>.zip`

#### macOS Distribution

```bash
yarn dist-mac
```

What this command does internally:

1. **`./scripts/fix-mac-build.sh`** — Pre-build cleanup script that:
   - Kills any stuck PKG builder processes
   - Cleans temporary build files
   - Fixes file permissions on `out/` and `dist/`
   - Clears Electron/electron-builder caches
2. **`npm run build`** — Runs typecheck + Vite build
3. **`electron-builder --mac pkg --x64`** — Creates x64 (Intel) `.pkg` installer:
   - Bundles **macOS JRE** and **TinyTeX-mac**
   - Applies entitlements from `entitlements.mac.plist`
   - Runs `electron-builder-hooks.cjs` to inject pre/post install scripts into the `.pkg`
4. **`electron-builder --mac pkg --arm64`** — Creates arm64 (Apple Silicon) `.pkg` installer
5. **`node scripts/package-build.cjs macos pre`** — Zips both `.pkg` files

**Output**: `dist/Criterion-<version>-x64.pkg` + `dist/Criterion-<version>-arm64.pkg` + `dist/macos-build-pre-<version>.zip`

> **Why two separate builds for macOS?** macOS supports both Intel (`x64`) and Apple Silicon (`arm64`) processors. Each architecture needs its own native binary. The build creates separate `.pkg` installers for each.

### Environment-Specific Builds

The project supports three environments. The environment affects which `.env.*` file is loaded and what label appears in the build artifact:

| Environment     | Env Suffix | `.env.*` File      | Use Case                                                    |
| --------------- | ---------- | ------------------ | ----------------------------------------------------------- |
| **Development** | `:dev`     | `.env.development` | Internal testing — debug tools enabled, verbose logging     |
| **Staging**     | `:staging` | `.env.staging`     | Pre-release testing — debug tools enabled, moderate logging |
| **Production**  | _(none)_   | `.env.production`  | Final release — debug tools disabled, error-only logging    |

**Examples:**

```bash
yarn dist-win           # Production Windows build
yarn dist-win:dev       # Development Windows build
yarn dist-win:staging   # Staging Windows build
```

### Build Artifacts & Output

After a distribution build, the `dist/` folder contains:

```
dist/
├── Criterion-<version>.exe              # Windows installer (if dist-win)
├── Criterion-<version>.deb              # Linux installer (if dist-linux)
├── Criterion-<version>-x64.pkg          # macOS Intel installer (if dist-mac)
├── Criterion-<version>-arm64.pkg        # macOS Apple Silicon installer (if dist-mac)
├── <platform>-build-<env>-<version>.zip # Zipped artifact for distribution
├── *.blockmap                           # Delta update metadata (auto-update)
└── builder-effective-config.yaml        # Resolved electron-builder config (for debugging)
```

> **Where does the version number come from?** It's read from the `version` field in `package.json` (currently `1.4.0`). To release a new version, update this field before building.

### CI/CD Pipeline

The GitLab CI/CD pipeline (`.gitlab-ci.yml`) automates builds on push to specific branches:

| Stage             | Branch                   | Jobs                                                                        |
| ----------------- | ------------------------ | --------------------------------------------------------------------------- |
| **Build-Check**   | `develop`, `pre`, `prod` | Typecheck (node + web), SonarQube analysis (prod only)                      |
| **Build-Windows** | `develop`, `pre`, `prod` | Windows `.exe` build using `electronuserland/builder:wine` Docker image     |
| **Build-Linux**   | `develop`, `pre`, `prod` | Linux `.deb` build using `electronuserland/builder` Docker image            |
| **Build-macOS**   | _(commented out)_        | Requires a macOS runner (not available in Docker) — currently done manually |

**Branch → Environment mapping in CI:**

| Branch    | CI Environment | Build command        |
| --------- | -------------- | -------------------- |
| `develop` | Development    | `yarn build:dev`     |
| `pre`     | Staging        | `yarn build:staging` |
| `prod`    | Production     | `yarn build`         |

> **macOS builds**: Since macOS cannot run in Docker, macOS builds are performed **manually** on a developer's machine using `yarn dist-mac`, `yarn dist-mac:dev`, or `yarn dist-mac:staging`.

### Pre-Commit Checklist

```bash
# ALWAYS run these checks before committing changes
yarn typecheck    # TypeScript type checking (node + web configs)
yarn lint         # ESLint checks
yarn test         # All Jest tests
```

### Troubleshooting Distribution Builds

| Symptom                                 | Cause                                   | Fix                                                                            |
| --------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------ |
| Build fails at typecheck                | TypeScript errors in codebase           | Fix all TS errors — build aborts on type errors                                |
| `electron-builder` hangs (macOS)        | Stuck PKG processes from previous build | Run `./scripts/fix-mac-build.sh` manually, then retry                          |
| `.pkg` missing pre/post install scripts | `scripts/pkg-scripts/` missing          | Check that `preinstall` and `postinstall` exist in `scripts/pkg-scripts/`      |
| Windows build fails on macOS/Linux      | Cross-compilation missing Wine          | CI uses `electronuserland/builder:wine` — for local cross-builds, install Wine |
| `dist/` empty after build               | Build step failed silently              | Check terminal output; run `yarn build` separately to isolate                  |
| Installer too large                     | Unnecessary resources bundled           | Check `electron-builder.json` `files` and `extraResources` filters             |
| Code signing error (macOS)              | Certificate not set                     | Set `CSC_LINK` and `CSC_KEY_PASSWORD` env vars                                 |
| `EPERM` or permission errors            | Previous build left locked files        | Run `yarn clean` then retry                                                    |

---

## 12. Coding Standards

### File Naming

| Type                        | Convention         | Example            |
| --------------------------- | ------------------ | ------------------ |
| React components (reusable) | `snake_case.tsx`   | `button_group.tsx` |
| React components (pages)    | `PascalCase.tsx`   | `HomePage.tsx`     |
| Hooks                       | `camelCase.ts`     | `useWindowSize.ts` |
| Utilities                   | `camelCase.ts`     | `formatDate.ts`    |
| Types                       | `PascalCase.ts`    | `DocumentTypes.ts` |
| Constants                   | `UPPER_SNAKE_CASE` | `MAX_FILE_SIZE`    |

### TypeScript Rules

```typescript
// DO: Use interfaces for object shapes
interface DocumentProps {
  title: string;
  content: JSONContent;
}

// DON'T: Use enums
enum Status { Draft, Published }  // Bad

// DO: Use string unions
type Status = 'draft' | 'published';  // Good

// DO: Functional components only
function MyComponent({ title }: Props) { ... }

// DON'T: Class components
class MyComponent extends React.Component { ... }  // Never
```

### React Patterns

```typescript
// Memoize expensive components
const ExpensiveList = React.memo(({ items }: Props) => { ... });

// useCallback for handlers passed to children
const handleClick = useCallback(() => {
  doSomething();
}, [dependency]);

// useMemo for expensive computations
const processed = useMemo(() =>
  expensiveOperation(data),
  [data]
);

// Avoid excessive useEffect
// Prefer derived state and event handlers
```

### Import Aliases

```typescript
// Configured in tsconfig.json and electron.vite.config.ts
import { Button } from "@components/ui/button";
import { useEditor } from "@pages/editor/provider";
import { formatDate } from "@utils/formatDate";
import { store } from "@store/store";
```

| Alias          | Path                                 |
| -------------- | ------------------------------------ |
| `@/`           | `src/renderer/src/`                  |
| `@components/` | `src/renderer/src/components/`       |
| `@pages/`      | `src/renderer/src/pages/`            |
| `@store/`      | `src/renderer/src/store/`            |
| `@utils/`      | `src/renderer/src/utils/`            |
| `@icons/`      | `src/renderer/src/components/icons/` |
| `@resources/`  | `buildResources/`                    |

### Code Comments

```typescript
// TODO: Implement feature X — issue #123
// FIXME: Memory leak when — issue #456
// @REFACTOR: This needs cleanup
// @MISSING: Not yet implemented

/**
 * Saves document to the specified path.
 * @param doc - The document to save
 * @param path - Destination file path
 * @returns Promise resolving to saved file path
 * @throws {SaveError} When write fails
 */
async function saveDocument(doc: Document, path: string): Promise<string> {
  ...
}
```

### Formatting (Prettier)

| Rule            | Value                        |
| --------------- | ---------------------------- |
| Indent          | 2 spaces                     |
| Semicolons      | Always                       |
| Quotes          | Single (JS/TS), Double (JSX) |
| Trailing comma  | Yes                          |
| Max line length | 100 chars                    |

---

## 13. Debugging Procedures

### Development Tools Configuration

```bash
# Development with full DevTools
yarn dev

# Enable DevTools in any environment
# Set VITE_ENABLE_DEV_TOOLS=true in .env file
```

### Diagnostic Procedures for Common Issues

#### IPC Communication Failures

```typescript
// 1. Check channel name matches exactly
// Main:
ipcMain.handle('document:save', ...)
// Preload:
ipcRenderer.invoke('document:save', ...)  // Must match!

// 2. Check preload is exposing the API
contextBridge.exposeInMainWorld('doc', api);

// 3. Check TypeScript declarations
// src/preload/index.d.ts must declare window.doc
```

#### Editor Content Not Saving

```typescript
// 1. Check content is valid JSONContent
const content = editor.getJSON();
console.log("Content:", JSON.stringify(content, null, 2));

// 2. Check Redux is updating
// Use Redux DevTools extension

// 3. Check IPC is called
window.doc.setMainTextContent(content).then(console.log);
```

#### Memory Leak Prevention

```typescript
// Documented causes and remediation:
// 1. TipTap editor not destroyed
useEffect(() => {
  return () => editor?.destroy(); // Always cleanup!
}, []);

// 2. Event listeners not removed
useEffect(() => {
  window.addEventListener("resize", handler);
  return () => window.removeEventListener("resize", handler);
}, []);

// 3. IPC listeners accumulating
// Use ipcMain.handle (not .on) for request-response
```

### Logging

```typescript
// Renderer → Main process logging
window.system.log("info", "Message here", additionalData);
window.system.log("error", "Error details", errorObject);
window.system.log("debug", "Debug info");
window.system.log("warn", "Warning message");

// Log levels: debug, info, warn, error
```

---

## 14. Contribution Guidelines

### Branch Strategy

```
main              # Production releases
  └── develop     # Integration branch
        ├── feature/ISSUE-123-description
        ├── fix/ISSUE-456-bug-description
        └── refactor/ISSUE-789-area
```

### Commit Convention

```
type(scope): description

feat(editor): add apparatus linking
fix(export): correct TEI namespace
refactor(store): simplify document slice
docs(readme): update setup instructions
test(tiptap): add lemma extension tests
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `style`

### Pull Request Checklist

- [ ] `yarn typecheck` passes
- [ ] `yarn lint` passes
- [ ] `yarn test` passes
- [ ] Self-reviewed the diff
- [ ] Updated relevant documentation
- [ ] Added tests for new features

### Implementation Example: Adding a New Feature

**Scenario**: Implementing a word count feature

1. **Create the UI component**:

```typescript
// src/renderer/src/components/word_count.tsx
export function WordCount({ count }: { count: number }) {
  return <span className="text-sm text-muted">{count} words</span>;
}
```

2. **Add Redux state** (if needed):

```typescript
// src/renderer/src/pages/editor/store/editor/editor.slice.ts
// Add to state interface and reducers
```

3. **Connect to editor**:

```typescript
// Use TipTap's storage or derive from content
const wordCount = useMemo(
  () => countWords(editor?.getText() ?? ""),
  [editor?.getText()],
);
```

4. **Add IPC channel** (if main process needed):

```typescript
// Main: ipcMain.handle('document:getWordCount', ...)
// Preload: wordCount: () => ipcRenderer.invoke('document:getWordCount')
```

5. **Write tests**:

```typescript
// src/renderer/src/components/word_count.test.tsx
describe('WordCount', () => {
  it('displays count correctly', () => { ... });
});
```

6. **Update types**:

```typescript
// src/preload/index.d.ts if adding new IPC
```

---

## 15. Appendices

### A. Glossary

| Term                   | Definition                                                        |
| ---------------------- | ----------------------------------------------------------------- |
| **Apparatus criticus** | Scholarly footnotes documenting textual variants                  |
| **Lemma**              | The word/phrase from the main text being annotated                |
| **Siglum** (pl. sigla) | Abbreviated manuscript identifier (e.g., "A", "B", "Par.lat.123") |
| **Reading**            | A textual variant found in a manuscript                           |
| **Witness**            | A manuscript that contains a version of the text                  |
| **TEI**                | Text Encoding Initiative — XML standard for scholarly texts       |
| **Collation**          | Comparing manuscripts to identify variants                        |

### B. Environment Variables

| Variable                | Description                   | Default       |
| ----------------------- | ----------------------------- | ------------- |
| `VITE_APP_ENV`          | Environment identifier        | `development` |
| `VITE_DEBUG_MODE`       | Enable debug features         | `false`       |
| `VITE_LOG_LEVEL`        | Log verbosity                 | `info`        |
| `VITE_ENABLE_DEV_TOOLS` | Show Electron DevTools        | `false`       |
| `CSC_LINK`              | Code signing certificate path | —             |
| `CSC_KEY_PASSWORD`      | Certificate password          | —             |

### C. IPC Channel Registry

<details>
<summary>Click to expand full IPC channel list</summary>

**Document Operations** (`document:*`):

- `document:save`, `document:saveAs`, `document:saveDocument`, `document:loadDocument`
- `document:getContent`, `document:setContent`
- `document:getApparatusesTypes`, `document:getApparatusNotes`, `document:setApparatusNotes`
- `document:getApparatusContent`, `document:setApparatusContent`
- `document:printPdf`, `document:savePdf`, `document:exportToTei`
- `document:getDocumentStyles`, `document:setDocumentStyles`, `document:importStyles`, `document:exportStyles`
- `document:getSigla`, `document:setSigla`, `document:importSigla`, `document:exportSigla`
- `document:find`, `document:replace`, `document:replaceAll`
- `document:getMetadata`, `document:setMetadata`
- `document:getPageSetup`, `document:setPageSetup`
- `document:getBibliography`, `document:setBibliography`, `document:importBibliography`

**Tab Management** (`tabs:*`):

- `tabs:new`, `tabs:close`, `tabs:select`, `tabs:reorder`
- `tabs:getSelectedTabId`, `tabs:getAllContentViewsIds`

**Menu Updates** (`menu:*`):

- `menu:enable`, `menu:disable`, `menu:update`
- `menu:setApparatusesMenu`, `menu:setRecentFilesMenu`
- `menu:enableUndo`, `menu:enableRedo`
- `menu:enableCut`, `menu:enableCopy`, `menu:enablePaste`

**System** (`system:*`):

- `system:getInstalledFonts`, `system:openFileDialog`, `system:saveFileDialog`
- `system:log`, `system:openExternal`
- `system:getAppVersion`, `system:getPlatform`, `system:getLocale`

**Application** (`application:*`):

- `application:setToolbarState`, `application:getToolbarState`
- `application:setStatusBar`, `application:getZoom`, `application:setZoom`

**Debug** (`debug:*`):

- `debug:getLayoutTabs`, `debug:getCurrentTabs`
- `debug:testTabRestoration`, `debug:forceSaveTabs`

</details>

### D. VS Code Configuration

**Recommended Extensions**:

- ESLint (`dbaeumer.vscode-eslint`)
- Prettier (`esbenp.prettier-vscode`)
- Tailwind CSS IntelliSense (`bradlc.vscode-tailwindcss`)
- ES7+ React Snippets (`dsznajder.es7-react-js-snippets`)

**Settings** (`.vscode/settings.json`):

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.preferences.importModuleSpecifier": "non-relative"
}
```

### E. External Resources

- [Electron Documentation](https://www.electronjs.org/docs)
- [React Documentation](https://react.dev)
- [TipTap Documentation](https://tiptap.dev)
- [Redux Toolkit](https://redux-toolkit.js.org)
- [TailwindCSS](https://tailwindcss.com/docs)
- [shadcn/ui](https://ui.shadcn.com)
- [TEI Guidelines](https://tei-c.org/guidelines/)

---

**Document Information**

| Field          | Value                            |
| -------------- | -------------------------------- |
| Last Updated   | March 2026                       |
| Document Owner | Criterion Development Team       |
| Classification | Internal Technical Documentation |
| Review Cycle   | Quarterly                        |
