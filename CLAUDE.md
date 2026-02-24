# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WebP Converter — a monorepo providing two tools for converting images to WebP format:
- **`packages/web`**: Browser-only web app (no server; all conversion via WebAssembly)
- **`packages/cli`**: Node.js CLI tool (`webp-convert`) using sharp for batch conversion

## Commands

```bash
# Root (runs across workspaces)
npm run dev          # Start web app dev server (Vite on :5173)
npm run build        # Build both packages
npm run test         # Run tests in all workspaces
npm run lint         # Biome lint
npm run format       # Biome format (write)
npm run check        # Biome check + auto-fix
npm run deploy       # Deploy web app to Cloudflare Workers (wrangler)

# Web package only
npm run dev -w @webp-converter/web
npm run test -w @webp-converter/web     # vitest run (jsdom)
npm run test:watch -w @webp-converter/web

# CLI package only
npm run build -w webp-convert
npm run test -w webp-convert            # vitest run
```

**Linting/formatting**: Biome (not ESLint/Prettier). Config at `biome.json`.

## Architecture

### `packages/web` — Browser-only React App

All image conversion runs entirely in the browser via **WebAssembly**. No network requests are made; no image data leaves the browser.

**Stack**: React 19 + Vite 7 + TypeScript + Tailwind CSS v4 + shadcn/ui (Radix UI)

**Key files:**
- `src/lib/converter.ts` — Core conversion logic. Lazily initializes @jsquash Wasm modules (singleton), decodes input image by MIME type, optionally resizes via `@jsquash/resize`, encodes to WebP. Supported inputs: JPEG, PNG, WebP, AVIF.
- `src/hooks/useConverter.ts` — All UI state (file, quality, scale, result, loading, error). Manages debounced re-conversion (400ms), AbortController for cancellation, and Blob URL lifecycle (revoke on change/unmount).
- `src/App.tsx` — Single-page layout. Renders `UploadArea` when no file selected, then switches to the workspace view (Preview + ControlPanel + InfoPanel + DownloadButton).
- `src/components/ui/` — shadcn/ui components (Button, Card, Slider, etc.)

**Wasm modules** (`@jsquash/*`) must be excluded from Vite's dep optimization (see `vite.config.ts`). Wasm files are handled as assets via `assetsInclude: ["**/*.wasm"]`.

**Deployment**: Cloudflare Workers (static assets mode). Config at `wrangler.jsonc` — points to `packages/web/dist`.

**Tests**: vitest + jsdom + @testing-library/react. Setup file: `src/test/setup.ts`. Path alias `@/` → `src/`.

### `packages/cli` — Node.js CLI Tool

**Stack**: Node.js ≥18 + TypeScript (built with tsup) + sharp + commander + fast-glob

**Entry**: `src/index.ts` (commander setup) → `src/converter.ts` (conversion logic) + `src/glob.ts` (input path resolution)

**Key behavior:**
- Converts files sequentially with individual error handling (not in parallel)
- `--dry-run` flag: processes without writing output files
- Output path resolution in `resolveOutputPath()`: defaults to same directory as input; if `--output` ends with `.webp`, treats as file path; otherwise treats as directory
- Batch conversions print a summary with total size savings
