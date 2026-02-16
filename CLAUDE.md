# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Azure DevOps extension for [microsoft/sbom-tool](https://github.com/microsoft/sbom-tool). It generates SPDX 2.2/2.3 compatible Software Bill of Materials (SBOMs) from Azure DevOps pipeline artifacts and displays them in a custom build results tab.

## Build Commands

```bash
# Install all dependencies (root + subdirectories)
npm install

# Build task and UI
npm run build

# Build individually
npm run build:task    # task/ -> task/dist/task/
npm run build:ui      # ui/ -> ui/dist/ui/

# Run tests
npm test

# Format code
npm run format
npm run format:check

# Package extension (creates VSIX in dist/)
npm run package
npm run package:dev   # Uses vss-extension.overrides.dev.json
npm run package:prod  # Uses vss-extension.overrides.prod.json
```

## Architecture

The project is structured as three npm packages with shared concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│  task/ - Azure Pipelines Task (Node.js)                         │
│  - Runs on build agents, invokes microsoft/sbom-tool            │
│  - Downloads sbom-tool if not present (brew/winget or direct)   │
│  - Post-processes SPDX JSON: adds security advisories,          │
│    generates XLSX spreadsheets and SVG graphs                   │
│  - Entry: task/index.ts -> SbomTool.generateAsync()             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  shared/ - Common TypeScript (used by both task and UI)         │
│  - spdx/models/2.3/ - SPDX 2.3 type definitions                  │
│  - spdx/convertSpdxToXlsx.ts - SPDX → Excel workbook            │
│  - spdx/convertSpdxToSvg.ts - SPDX → network graph SVG          │
│  - ghsa/client.ts - GitHub Security Advisory GraphQL client     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  ui/ - Azure DevOps UI Extension (React)                        │
│  - Build results tab showing SBOM packages, licenses,           │
│    security advisories, and relationship graphs                 │
│  - Entry: ui/sbom-report-tab.tsx                                │
│  - Uses azure-devops-extension-sdk and azure-devops-ui          │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Flow

1. Pipeline task runs `sbom-tool generate` → produces SPDX JSON
2. Task optionally enriches SPDX with GitHub security advisories via GraphQL
3. Task attaches SPDX JSON and optional SVG to build timeline
4. UI extension reads attached files and renders interactive tables/charts

### Important Files

- `task/task.json` - Azure Pipelines task definition (inputs, execution)
- `vss-extension.json` - Azure DevOps extension manifest
- `shared/ghsa/client.ts` - GitHub Advisory Database integration (requires PAT)

## Development Notes

### Testing

Tests use Jest with ts-jest. Test files use `.test.ts` suffix:

```bash
npm test                           # Run all tests
npx jest shared/spdx/convertSpdxToXlsx.test.ts  # Run specific test
```

### Local UI Development

```bash
npm run start:ui  # Starts webpack-dev-server on https://localhost:3000
```

The UI requires running in an Azure DevOps context. For local testing, update `vss-extension.json` baseUri to point to localhost.

### Extension Packaging

The extension is packaged using `tfx-cli`. Before packaging, ensure both task and UI are built:

```bash
npm run build
npm run package
```

This increments the version and creates a VSIX in `dist/`.

### Adding New Task Inputs

1. Add input definition to `task/task.json`
2. Read input in `task/index.ts` using `getInput()` or `getBoolInput()`
3. Pass to `SbomTool.generateAsync()` via `SbomToolGenerateArgs` interface
4. Update `task/utils/sbomTool.ts` to handle the new argument
