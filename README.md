# Criterion

Criterion is a professional desktop application designed for critical text edition: reconstructing historical texts from multiple manuscript sources, documenting textual variants, and producing annotated critical editions.

This repository contains:
- A desktop application built with Electron, React, and TypeScript, using Tiptap for editing.
- A print-preview component implemented in Java, which generates LaTeX and compiles it with XeLaTeX via a TinyTeX toolchain.

## Scope and limitations

In scope:
- Philological workbench for scholarly research
- Critical apparatus (apparatus criticus) editing
- Print-oriented workflows for critical editions
- Offline-first desktop operation

Out of scope:
- General-purpose word processing
- Real-time collaborative editing
- Web-based or browser deployment
- Cloud-dependent service architecture

## Export formats

Project documentation describes export capabilities that include:
- PDF (print-oriented output)
- XML/TEI (TEI P5)

## Platform support

Project documentation describes production packaging for:
- macOS (x64, arm64)
- Windows (x64)
- Linux (x64)

Treat published releases (and the repository build configuration) as the authoritative source of what is currently shipped.

## Repository structure (high level) (https://code-repo.d4science.org/Resilience/WP3-criterion/)

A high-level structure is documented as:

- src/
  - main/ (Electron main process)
  - preload/ (contextBridge API surface)
  - renderer/src/ (React UI)
- i18n/ (translations)
- resources/ (JSON configs)
- buildResources/ (build assets, including print preview assets)
- docs/ (documentation)

## Development setup (desktop application)

Follow the repository configuration and developer documentation as the source of truth for prerequisites and commands (for example: .nvmrc, package.json, yarn.lock, and any developer guide in docs/).

The typical baseline workflow is:

# Step 1: Clone the repository
git clone <repository-url> criterion && cd criterion

# Step 2: Configure Node.js runtime (via nvm)
nvm install && nvm use

# Step 3: Enable Yarn package manager
corepack enable

# Step 4: Install project dependencies
yarn install

# Step 5: Launch development server
yarn dev

## Print preview component (Java + TinyTeX)

The print-preview component expects a specific runtime folder layout.

Safety note:
- The print-preview tool treats its compile directory as disposable and may delete files in that directory during normal operation. Do not point it at a folder containing anything important.

Expected runtime layout (conceptual):

<APP_HOME>/
  PrintPreviewNav.jar
  fonts/
  TinyTeX-win/… (Windows)
  TinyTeX-lin/… (Linux)
  TinyTeX-mac/… (macOS)

For CLI usage, end-to-end examples, and build guidance, follow the print-preview developer documentation included in this repository.

## Build and distribution

Build and distribution must be executed using the repository’s build scripts and configuration (CI and package.json scripts are authoritative). Only claim platform support for artifacts that are actually produced and published.

## Governance and project processes

Project governance and contribution rules are documented in:
- GOVERNANCE.md
- COMMUNITY_PROCESSES.md
- CoC.md
- CONTRIBUTING.md
- SECURITY.md
- RELEASE.md
- SBOM.md
- RFC.md
- ADR.md

Decision records (if present in your checkout):
- RFCs: docs/rfcs/
- ADRs: docs/adrs/

If a directory or document is not present in your checkout, do not invent an alternative location. Propose changes via the governance process.

## Contributing

Read CONTRIBUTING.md before opening issues or pull requests. It defines:
- contribution workflow and expectations
- review and quality gates
- when RFCs/ADRs are required for larger changes

## Security

Do not disclose suspected vulnerabilities in public issues, pull requests, or discussions. Follow SECURITY.md for confidential reporting and coordinated disclosure.

## License

See the license files included in this repository (including any platform-specific license agreement documents, if applicable).

## Acknowledgements

See the acknowledgements document(s) included in this repository.
