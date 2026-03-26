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

### Official References

Use the official project pages below when setting up the environment:

- **Git clone workflow**: [GitHub Docs — Cloning a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)
- **nvm (macOS/Linux)**: [nvm-sh/nvm — Git install](https://github.com/nvm-sh/nvm#git-install)
- **nvm verification and troubleshooting**: [nvm-sh/nvm — Verify installation](https://github.com/nvm-sh/nvm#verify-installation)
- **nvm-windows**: [coreybutler/nvm-windows — latest releases](https://github.com/coreybutler/nvm-windows/releases/latest)
- **Visual Studio Build Tools**: [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/)
- **Corepack**: [nodejs/corepack — installation and usage](https://github.com/nodejs/corepack#how-to-install)
- **Yarn**: [Yarn installation guide](https://yarnpkg.com/getting-started/install)

### Recommended Setup Path

For the least error-prone setup, follow this sequence exactly:

1. Install platform build tools
2. Install `nvm`
3. Run `nvm install && nvm use`
4. Run `corepack enable && corepack install`
5. Run `yarn install`

Avoid installing Node.js or Yarn globally for this project unless you have a specific reason to bypass the repository-managed toolchain.

> **Why nvm instead of installing Node.js directly?**  
> The project pins an exact Node.js version in the `.nvmrc` file (currently `24.11.0`). Using nvm guarantees every developer runs the same version and avoids "works on my machine" issues. When you run `nvm install && nvm use`, nvm reads `.nvmrc` and installs/activates that exact version automatically.

> **Why Yarn 4 Berry?**  
> Yarn 4 Berry is faster and more deterministic than npm or Yarn 1. This repository pins Yarn through the `packageManager` field in `package.json` (`yarn@4.12.0`) and configures it via `.yarnrc.yml` with `nodeLinker: node-modules`, meaning it still creates a traditional `node_modules/` folder (some Electron tooling requires this).

### Platform-Specific Setup

You **must** complete the platform-specific setup for your operating system **before** running the installation steps.

<details>
<summary><strong>macOS</strong></summary>

```bash
# 1. Install Xcode Command Line Tools (required for compiling native modules)
#    This installs gcc, make, and other build essentials.
#    You do NOT need the full Xcode.app to build Criterion.
xcode-select --install

# 2. Install nvm using the official git-based method
git clone https://github.com/nvm-sh/nvm.git ~/.nvm
cd ~/.nvm
git checkout v0.40.4

# 3. Add nvm to your shell profile (run once, choose the file for your shell)
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.zshrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.zshrc
# For bash, write the same two lines to ~/.bashrc instead.

# 4. Reload your shell so the `nvm` command becomes available
source ~/.zshrc    # If you use zsh (default on modern macOS)
# source ~/.bashrc # If you use bash instead
```

**Verify it worked:**

```bash
command -v nvm  # Should print: nvm
nvm --version   # Should print a version number like 0.40.4
```

Official references:

- [nvm-sh/nvm — Git install](https://github.com/nvm-sh/nvm#git-install)
- [nvm-sh/nvm — Verify installation](https://github.com/nvm-sh/nvm#verify-installation)

</details>

<details>
<summary><strong>Ubuntu/Linux</strong></summary>

```bash
# 1. Install system dependencies (required by Electron and native modules)
#    These libraries are needed for Electron's Chromium runtime to function correctly.
sudo apt update
sudo apt install -y git build-essential \
  libgtk-3-dev libnotify-dev libnss3 libxss1 libasound2 \
  libatk-bridge2.0-0 libgdk-pixbuf2.0-0 libx11-xcb1 \
  libxcomposite1 libxdamage1 libxrandr2 libxcursor1 \
  libxext6 libxcb1 libnspr4 libdrm2 libcups2 libexpat1

# 2. Install nvm using the official git-based method
git clone https://github.com/nvm-sh/nvm.git ~/.nvm
cd ~/.nvm
git checkout v0.40.4

# 3. Add nvm to your shell profile (run once)
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.bashrc
# If you use zsh on Linux, write the same two lines to ~/.zshrc instead.

# 4. Reload your shell
source ~/.bashrc
```

**Verify it worked:**

```bash
command -v nvm  # Should print: nvm
nvm --version   # Should print a version number like 0.40.4
```

Official references:

- [nvm-sh/nvm — Git install](https://github.com/nvm-sh/nvm#git-install)
- [nvm-sh/nvm — Verify installation](https://github.com/nvm-sh/nvm#verify-installation)

> **Linux Sandbox Note**: If you encounter sandbox errors when running the app, use `yarn dev-linux` instead of `yarn dev`. This sets `ELECTRON_DISABLE_SANDBOX=1` to bypass Chrome sandbox restrictions on some Linux configurations.

</details>

<details>
<summary><strong>Windows</strong></summary>

1. **Install nvm-windows**: Download the installer from [coreybutler/nvm-windows — latest releases](https://github.com/coreybutler/nvm-windows/releases/latest) and run it
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

Once your platform-specific setup is complete, run the following steps in order. Make sure you run those commands with administration rights.

1. Clone the repository

```bash
git clone https://code-repo.d4science.org/Resilience/WP3-criterion.git criterion && cd criterion
```

Expected result: you are inside the `criterion` project folder.

2. Install and activate the required Node.js version

```bash
nvm install 24.11.0
nvm use 24.11.0
```

Expected result: terminal shows Node.js `v24.11.0` in use.

3. Enable Corepack and provision the project Yarn version

```bash
corepack enable
corepack install
```

Expected result: commands exit without errors and `yarn --version` resolves to the version pinned by the repository.

4. Install dependencies

```bash
yarn install
```

Expected result: dependencies are installed into `node_modules`.

5. Create a local `.env` file

macOS/Linux:

```bash
cat > .env <<'EOF'
# Criterion - Development Environment
# =============================================================================

# Environment identifier
VITE_APP_ENV=development

# =============================================================================
# Document Signing (Internal)
# =============================================================================
MAIN_VITE_DOCUMENT_SIGN_KEY=81f1cc5b1d262b8e5d028dc6076dded7783e4990e670dfe0934d125c25e0073a

# =============================================================================
# Debug & Logging
# =============================================================================
VITE_DEBUG_MODE=true
VITE_LOG_LEVEL=debug

# =============================================================================
# Feature Flags
# =============================================================================
VITE_ENABLE_DEV_TOOLS=true
VITE_ENABLE_MOCK_DATA=false
EOF
```

Windows PowerShell:

```powershell
@"
# Criterion - Development Environment
# =============================================================================

# Environment identifier
VITE_APP_ENV=development

# =============================================================================
# Document Signing (Internal)
# =============================================================================
MAIN_VITE_DOCUMENT_SIGN_KEY=81f1cc5b1d262b8e5d028dc6076dded7783e4990e670dfe0934d125c25e0073a

# =============================================================================
# Debug & Logging
# =============================================================================
VITE_DEBUG_MODE=true
VITE_LOG_LEVEL=debug

# =============================================================================
# Feature Flags
# =============================================================================
VITE_ENABLE_DEV_TOOLS=true
VITE_ENABLE_MOCK_DATA=false
"@ | Set-Content -Path .env
```

> **Important — Document signing key compatibility**
> `MAIN_VITE_DOCUMENT_SIGN_KEY` is used to sign documents.
> If this key changes, documents created with the previous key will no longer be usable.
> Only documents created after the key change will be usable.

6. Launch the development server

```bash
yarn dev
```

Linux sandbox fallback:

```bash
yarn dev-linux
```

Expected result: Vite compiles all processes and Electron opens the application window.

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
| `.env`                    | Local environment variables for development (`VITE_DEBUG_MODE=true`, `VITE_LOG_LEVEL=debug`, etc.).                                                                   |
| `eslint.config.mjs`       | ESLint 9 flat config for linting rules.                                                                                                                                |
| `.stylelintrc.json`       | Stylelint config for CSS/SCSS linting.                                                                                                                                 |
| `tailwind.config.cjs`     | TailwindCSS configuration with custom color palette.                                                                                                                   |
| `postcss.config.cjs`      | PostCSS config (used by TailwindCSS).                                                                                                                                  |
| `jest.config.js`          | Jest test runner configuration.                                                                                                                                        |
| `components.json`         | shadcn/ui component configuration.                                                                                                                                     |

### Environment Variables Quick Reference

Use a single local `.env` file for development.

| Variable                      | `.env` value    |
| ----------------------------- | --------------- |
| `VITE_APP_ENV`                | `development`   |
| `VITE_DEBUG_MODE`             | `true`          |
| `VITE_LOG_LEVEL`              | `debug`         |
| `VITE_ENABLE_DEV_TOOLS`       | `true`          |
| `VITE_ENABLE_MOCK_DATA`       | `false`         |
| `MAIN_VITE_DOCUMENT_SIGN_KEY` | _(set manually)_ |

> **Security note**: Keep `.env` local only. Never commit real signing keys to the repository.

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
└── *.md                         # Project documentation at repository root
```

### Strict Boundaries

| Layer       | Can Import From                                      | Should Not Import From                  |
| ----------- | ---------------------------------------------------- | --------------------------------------- |
| `main/`     | Node.js, Electron main APIs, app main/shared modules | `renderer/`, preload UI-facing code     |
| `preload/`  | Electron preload APIs, shared types/utilities        | `renderer/`, main window/menu internals |
| `renderer/` | React, browser APIs, `window.*` preload APIs         | `main/`, Node.js built-ins              |

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
│ React Component  │  calls window.doc.saveDocument()
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Preload Bridge   │  ipcRenderer.invoke('document:saveDocument')
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Main Process     │  ipcMain.handle('document:saveDocument', handler)
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

The main document views and most child windows use the following security settings. Helper windows such as the loader and tooltip use different settings because they render inline HTML without a preload bridge.

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
  document:saveDocument
  document:export:tei
  tabs:new
  menu:update:apparatuses
```

---

## 6. Preload API Reference

The preload script currently exposes `window.electron` from `@electron-toolkit/preload` plus **10 application-specific namespaces** via `window.*`.

Source of truth: `src/preload/index.ts`.

### Namespace Overview

| Namespace                  | Purpose                           | Method Count |
| -------------------------- | --------------------------------- | ------------ |
| `window.tabs`              | Tab lifecycle management          | 6            |
| `window.menu`              | Menu state updates                | 19           |
| `window.system`            | System utilities and logging      | 9            |
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
window.doc.openDocument(): Promise<void>
window.doc.openDocumentAtPath(filePath: string): Promise<void>
window.doc.saveDocument(): Promise<boolean>

// Content
window.doc.getMainText(): Promise<JSONContent | null>
window.doc.setMainText(content: JSONContent | null, shouldMarkAsTouched?: boolean): Promise<void>

// Apparatuses
window.doc.getApparatuses(): Promise<DocumentApparatus[]>
window.doc.getApparatusWithId(id: string): Promise<DocumentApparatus | undefined>
window.doc.updateApparatusIdWithContent(id: string, content: JSONContent, shouldMarkAsTouched?: boolean): Promise<void>

// Export
window.doc.print(includeContent: PrintIncludeContents, options?: PrintOptions): Promise<void>
window.doc.exportToTei(): Promise<void>
window.doc.savePdf(includeSections: PrintSections): Promise<void>
window.doc.getPrintPreview(): Promise<{ path: string | null; isLoaded: boolean; error: string | null }>

// Styles & Templates
window.doc.getStyles(): Promise<Style[]>
window.doc.setStyles(styles: object[]): Promise<void>
window.doc.importStyles(): Promise<string | null>
window.doc.exportStyles(): Promise<void>
window.doc.getTemplate(): Promise<Template>
window.doc.getTemplates(): Promise<{ filename: string; template: Template }[]>

// Sigla
window.doc.getSiglumList(): Promise<DocumentSiglum[]>
window.doc.setSiglumList(sigla: DocumentSiglum[] | null): Promise<void>
window.doc.importSigla(): Promise<DocumentSiglum[]>
window.doc.exportSigla(): Promise<void>

// Find & Replace
window.doc.openFind(): Promise<void>
window.doc.findNext(): Promise<void>
window.doc.findPrevious(): Promise<void>
window.doc.replace(replacement: string): Promise<void>
window.doc.replaceAll(replacement: string): Promise<void>
window.doc.setSearchCriteria(options: SearchCriteria): Promise<void>
```

#### `window.system` — System Utilities

```typescript
window.system.getUserInfo(): Promise<void>
window.system.getExternalLinks(): Promise<ExternalLinks>
window.system.getFonts(): Promise<string[]>
window.system.getSubsets(): Promise<Subset[]>
window.system.getSymbols(fontName: string): Promise<CharacterSet>
window.system.getConfiguredSpcialCharactersList(): Promise<CharacterConfiguration[]>
window.system.showMessageBox(title: string, message: string, buttons: string[], type?: string): Promise<Electron.MessageBoxReturnValue>
window.system.findWorker(payload: WorkerRequest): Promise<WorkerMatch[]>
window.system.log(entry: LogEntry): Promise<void>
```

#### `window.tabs` — Tab Management

```typescript
window.tabs.new(fileType: FileType): Promise<number | null>
window.tabs.close(id: number): Promise<void>
window.tabs.select(id: number, tabType: TabType): Promise<void>
window.tabs.reorder(tabIds: number[]): Promise<void>
window.tabs.getSelectedTabId(): Promise<number>
window.tabs.getAllContentViewsIds(): Promise<number[]>
```

#### `window.debug` — Debug Utilities

```typescript
window.debug.getLayoutTabs(): Promise<Tab[]>
window.debug.getCurrentTabs(): Promise<Tab[]>
window.debug.testTabRestoration(): Promise<{ success: boolean; count?: number; error?: string }>
window.debug.forceSaveTabs(): Promise<Tab[]>
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

// Persist via IPC
await window.doc.setMainText(content, true);
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
  editorMode: 'editing' | 'review';
  canUndo: boolean;
  canRedo: boolean;
  apparatuses: Apparatus[];
  documentTemplate: Template | null;
  selectedNodeType: 'heading' | 'paragraph' | 'mixed' | null;
}

const initialState: EditorState = {
  editorMode: 'editing',
  canUndo: false,
  canRedo: false,
  apparatuses: [],
  documentTemplate: null,
  selectedNodeType: null,
};

export const editorSlice = createSlice({
  name: "editor",
  initialState,
  reducers: {
    setCanUndo: (state, action: PayloadAction<boolean>) => {
      state.canUndo = action.payload;
    },
    loadDocumentApparatuses: (state, action: PayloadAction<DocumentApparatus[]>) => {
      state.apparatuses = action.payload.map((apparatus) => ({
        id: apparatus.id,
        title: apparatus.title,
        type: apparatus.type,
        visible: apparatus.visible ?? true,
        expanded: apparatus.expanded ?? true,
        notesVisible: apparatus.notesVisible ?? true,
        commentsVisible: apparatus.commentsVisible ?? true,
      }));
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
  id: string;
  version: string;
  signature: string;

  // Content
  mainText: JSONContent | null;
  apparatuses: DocumentApparatus[];
  annotations: Annotations;

  // Configuration
  template: Template;
  referencesFormat: ReferencesFormat;
  metadata: Metadata;

  // References
  sigla: DocumentSiglum[];
  bibliographies: Bibliography[];
}
```

When loading a document, the main process normalizes snake_case and camelCase keys and may preserve additional fields during migration.

### App Settings Persistence

Application settings are stored via `electron-store`:

```typescript
// src/main/store.ts
import Store from "electron-store";

const store = new Store({
  defaults: {
    theme: "system",
    statusbarVisible: true,
    toolbarIsVisible: true,
    recentDocuments: [],
    statusBarConfig: ["pageNumber", "wordCount", "zoom"],
    zoom: "100",
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
│     node scripts/package-build.cjs <platform> dev  →  Zips the          │
│     installer artifact into a versioned .zip for distribution           │
│     Output: dist/<platform>-build-dev-<version>.zip                     │
└──────────────────────────────────────────────────────────────────────────┘
```

### Build Commands Reference

These commands **compile TypeScript → JavaScript** but do NOT create installers:

| Command              | Environment | What It Does                                           | Output        |
| -------------------- | ----------- | ------------------------------------------------------ | ------------- |
| `yarn build`         | Development | Runs typecheck + Vite build using local `.env`         | `out/` folder |

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
> **To obtain a new version of `printpreview-0.0.1-SNAPSHOT.jar`, refer to the Print Preview Developer Guide.** The JAR is produced by a separate Java project, and that guide explains how to build and regenerate the artifact.

> **How does electron-builder know what to include?** The `extraResources` section in `electron-builder.json` specifies which files from `buildResources/` are bundled. Each platform configuration (mac/win/linux) has its own `extraResources` block that includes only the JRE and TinyTeX for that operating system, keeping the installer size down.

### Distribution Commands — Full Reference

| Command                   | Platform        | Env         | Installer Output | Zip Output                    | Runs On |
| ------------------------- | --------------- | ----------- | ---------------- | ----------------------------- | ------- |
| `yarn dist-all`           | All             | Development | Mixed            | Per-platform `criterion_<platform>_<ver>.zip` | macOS only |
| `yarn dist-win`           | Windows x64     | Development | `.exe` (NSIS)    | `criterion_windows_<ver>.zip` | Windows only |
| `yarn dist-linux`         | Linux x64       | Development | `.deb`           | `criterion_linux_<ver>.zip`   | Linux only |
| `yarn dist-mac`           | macOS x64+arm64 | Development | `.pkg` (×2)      | `criterion_macos_<ver>.zip`   | macOS only |

### Step-by-Step Distribution Per Platform

#### Windows Distribution

```bash
yarn dist-win
```

What this command does internally:

1. **`cross-env NODE_ENV=development`** — Sets environment to development
2. **`npm run build`** — Runs typecheck + Vite build (creates `out/` folder)
3. **`electron-builder --win --x64`** — Packages everything into an NSIS installer:
   - Bundles compiled code from `out/`
  - Bundles `node_modules` needed by the build
   - Copies `extraResources` (i18n, buildResources, **Windows JRE**, **TinyTeX-win**)
   - Creates `dist/Criterion-<version>.exe`
4. **`node scripts/package-build.cjs windows dev`** — Zips the `.exe` into `dist/criterion_windows_<version>.zip`

**Output**: `dist/Criterion-<version>.exe` + `dist/criterion_windows_<version>.zip`

#### Linux Distribution

```bash
yarn dist-linux
```

What this command does internally:

1. **`cross-env NODE_ENV=development`** — Sets environment to development
2. **`npm run build`** — Runs typecheck + Vite build
3. **`electron-builder --linux --x64 --config.linux.target=deb`** — Creates a `.deb` package:
   - Includes `postinstall.sh` script (runs after installation on target machine)
   - Bundles **Linux JRE** and **TinyTeX-lin**
   - Declares system dependencies (libgtk-3-0, libnss3, etc.) in the `.deb` metadata
   - Creates `dist/Criterion-<version>.deb`
4. **`node scripts/package-build.cjs linux dev`** — Zips the `.deb` into `dist/criterion_linux_<version>.zip`

**Output**: `dist/Criterion-<version>.deb` + `dist/criterion_linux_<version>.zip`

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
5. **`node scripts/package-build.cjs macos dev`** — Zips both `.pkg` files into `dist/criterion_macos_<version>.zip`

**Output**: `dist/Criterion-installer-<version>-x64.pkg` + `dist/Criterion-installer-<version>-arm64.pkg` + `dist/criterion_macos_<version>.zip`

> **Why two separate builds for macOS?** macOS supports both Intel (`x64`) and Apple Silicon (`arm64`) processors. Each architecture needs its own native binary. The build creates separate `.pkg` installers for each.

### Development-Only Builds

This repository is configured for development-only build workflows.

**Examples:**

```bash
yarn dist-all
yarn dist-win
yarn dist-linux
yarn dist-mac
```

### Build Artifacts & Output

After a distribution build, the `dist/` folder contains:

```
dist/
├── Criterion-<version>.exe              # Windows installer (if dist-win)
├── Criterion-<version>.deb              # Linux installer (if dist-linux)
├── Criterion-installer-<version>-x64.pkg          # macOS Intel installer (if dist-mac)
├── Criterion-installer-<version>-arm64.pkg        # macOS Apple Silicon installer (if dist-mac)
├── criterion_<platform>_<version>.zip    # Zipped artifact created by scripts/package-build.cjs
├── *.blockmap                           # Delta update metadata (auto-update)
└── builder-effective-config.yaml        # Resolved electron-builder config (for debugging)
```

> **Where does the version number come from?** It's read from the `version` field in `package.json` (currently `1.1.0`). To release a new version, update this field before building.

### Pre-Commit Checklist

```bash
# Minimum required check before committing changes
yarn typecheck    # TypeScript type checking (node + web configs)
```

### Troubleshooting Distribution Builds

| Symptom                                 | Cause                                   | Fix                                                                            |
| --------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------ |
| Build fails at typecheck                | TypeScript errors in codebase           | Fix all TS errors — build aborts on type errors                                |
| `electron-builder` hangs (macOS)        | Stuck PKG processes from previous build | Run `./scripts/fix-mac-build.sh` manually, then retry                          |
| `.pkg` missing pre/post install scripts | `scripts/pkg-scripts/` missing          | Check that `preinstall` and `postinstall` exist in `scripts/pkg-scripts/`      |
| Windows build fails on macOS/Linux      | Cross-compilation support not available locally | Use a Windows environment, or install the toolchain required by your host setup |
| `dist/` empty after build               | Build step failed silently              | Check terminal output; run `yarn build` separately to isolate                  |
| Installer too large                     | Unnecessary resources bundled           | Check `electron-builder.json` `files` and `extraResources` filters             |
| `EPERM` or permission errors            | Previous build left locked files        | Run `yarn clean` then retry                                                    |

---

## 12. Coding Standards

### File Naming

| Type                        | Convention         | Example            |
| --------------------------- | ------------------ | ------------------ |
| Reusable React components   | Mixed: `PascalCase`, `kebab-case`, `snake_case` | `Typography.tsx`, `button-popover.tsx`, `account_button.tsx` |
| React views/pages           | Mostly `PascalCase.tsx` | `WelcomeView.tsx` |
| Hooks                       | Mostly `use-*.ts[x]` | `use-electron.ts`, `use-mobile.tsx` |
| Utilities                   | Mostly `camelCase.ts` | `clipboardUtils.ts` |
| Types                       | Global `.d.ts` and adjacent type files | `types.d.ts`, `shared/types.ts` |
| Constants                   | Mixed: `camelCase` for maps/config, `UPPER_SNAKE_CASE` for fixed values | `apparatusTypeName`, `DEFAULT_PRINT_SECTION_SELECTED`, `FIND_MAX_DEPTH` |

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
import cn from "@/utils/classNames";
import Button from "@components/ui/button";
import { editorContext } from "@pages/editor/provider/context";
import store from "@store/store";
import { textFormatColors } from "@utils/optionsEnums";
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
ipcMain.handle('document:saveDocument', ...)
// Preload:
ipcRenderer.invoke('document:saveDocument')  // Must match exactly!

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
window.doc.setMainText(content, true).then(console.log);
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
window.system.log({
  timestamp: new Date().toISOString(),
  level: "info",
  process: "renderer",
  category: "editor",
  message: "Message here",
});

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

### C. IPC Channel Registry

<details>
<summary>Click to expand a repository-verified IPC channel summary</summary>

**Document Operations** (`document:*`):

- `document:openDocument`, `document:openDocumentAtPath`, `document:saveDocument`
- `document:getMainText`, `document:setMainText`
- `document:getApparatuses`, `document:getApparatusWithId`, `document:updateApparatusIdWithContent`
- `document:getTemplate`, `document:getTemplates`, `document:setTemplate`, `document:importTemplate`, `document:createTemplate`
- `document:getStyles`, `document:getStylesFileNames`, `document:getStylesFromFile`, `document:importStyles`, `document:exportStyles`
- `document:getSiglumList`, `document:setSiglumList`, `document:importSigla`, `document:exportSigla`
- `document:print`, `document:savePdf`, `document:exportToTei`, `document:getPrintPreview`
- `document:openFind`, `document:findNext`, `document:findPrevious`, `document:replace`, `document:replaceAll`
- `document:setSearchCriteria`, `document:resetSearchCriteria`, `document:setReplaceInProgress`
- `document:getMetadata`, `document:setMetadata`
- `document:getPageSetup`, `document:setPageSetup`
- `document:getBibliographies`, `document:setBibliographies`, `document:importBibliography`

**Tab Management** (`tabs:*`):

- `tabs:new`, `tabs:close`, `tabs:select`, `tabs:reorder`
- `tabs:getSelectedTabId`, `tabs:getAllContentViewsIds`

**Menu Updates** (`menu:*`):

- `menu:disableReferencesMenuItems`, `menu:updateViewApparatusesMenuItems`
- `menu:setTocVisibility`, `menu:setLineNumberShowLines`, `menu:setPrintPreviewVisibility`
- `menu:setTocMenuItemsEnabled`, `menu:setTocSettingsEnabled`, `menu:setMenuFeatureEnabled`
- `menu:setAddCommentMenuItemEnabled`, `menu:setAddBookmarkMenuItemEnabled`, `menu:setAddNoteMenuItemEnabled`
- `menu:setAddReadingsEnabled`, `menu:setReferencesMenuCurrentContext`, `menu:setSiglumMenuItemEnabled`
- `menu:setLinkMenuItemEnabled`, `menu:setRemoveLinkMenuItemEnabled`, `menu:setAddCitationMenuItemEnabled`, `menu:setSymbolMenuItemEnabled`

**System** (`system:*`):

- `system:getUserInfo`, `system:getExternalLinks`
- `system:getFonts`, `system:getSubsets`, `system:getSymbols`
- `system:getConfiguredSpcialCharactersList`, `system:showMessageBox`, `system:worker`, `system:log`

**Application** (`application:*`):

- `application:toolbarIsVisible`, `application:getStatusBarVisibility`
- `application:readToolbarAdditionalItems`, `application:updateToolbarAdditionalItems`
- `application:readStatusBarConfig`, `application:storeStatusBarConfig`
- `application:readZoom`, `application:storeZoom`, `application:closeChildWindow`

**Preferences / Theme / Tooltip / Shortcuts**:

- `preferences:get`, `preferences:save`
- `pageSetup:get`, `pageSetup:save`
- `theme:setTheme`, `theme:getTheme`
- `tooltip:show`, `tooltip:hide`, `tooltip:set-text`
- `keyboard-shortcuts:getShortcuts`, `keyboard-shortcuts:setShortcut`, `keyboard-shortcuts:removeShortcut`, `keyboard-shortcuts:resetAll`

**Debug** (`debug:*`):

- `debug:getLayoutTabs`, `debug:getCurrentTabs`
- `debug:testTabRestoration`, `debug:forceSaveTabs`

For the full runtime-exposed surface, use `src/preload/index.ts` and `src/main/index.ts` as the source of truth.

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
