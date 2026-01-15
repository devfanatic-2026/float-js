# Pull Request: Robust ESM Module Resolution & SSR Asset Handling

This document outlines the proposed changes to the Float.js core transformation and SSR logic to address critical issues in module resolution and server-side rendering.

## Overview

The current on-the-fly transformation logic fails when TypeScript files import other TypeScript files, as Node.js cannot natively import `.ts` files. Additionally, JSON imports require explicit import assertions in Node.js ESM, and static assets (CSS, images) cause crashes during SSR.

## Proposed Changes

### 1. Recursive TypeScript Transformation in `transform.ts`

**Problem**: The original implementation transformed only the entry file to `.mjs`, but left imported TypeScript files as `.ts`, causing `ERR_UNKNOWN_FILE_EXTENSION` errors.

**Solution**:
- **Stable Cache-Based Transformation**: Each TypeScript file is transformed into a stable `.mjs` file in the `.float/.cache` directory using a content-based hash of its absolute path.
- **Recursive Processing**: When rewriting imports, the framework now detects `.ts`, `.tsx`, and `.jsx` files and recursively transforms them before updating the import path.
- **Cache Invalidation**: Files are re-transformed only if the source file has been modified (mtime check).
- **Query Parameter for Cache Busting**: Import URLs include a `?t=mtime` parameter to ensure Node.js reloads the module when the source changes.

### 2. JSON Import Assertions in `transform.ts`

**Problem**: Node.js ESM requires `assert { type: "json" }` for JSON imports, which esbuild doesn't add automatically.

**Solution**:
- **Post-Processing Step**: After all path rewrites are complete, a final regex pass detects all `.json` imports and adds the required assertion.
- **Idempotent**: The logic checks if an assertion already exists to avoid duplicates.

### 3. SSR Asset Mocking in `transform.ts`

**Problem**: Importing CSS, images, or other non-JS assets during SSR causes Node.js to attempt parsing them as JavaScript.

**Solution**:
- **Dummy Module Injection**: Detects imports of non-code assets (`.css`, `.png`, `.svg`, etc.) and replaces them with a data-URI returning a dummy ESM module (`export default {}`).

### 4. Path Alias Support

- **`@/` and `~/` Aliases**: The framework now resolves root-relative path aliases commonly used in modern projects.
- **Exhaustive Extension Resolution**: Automatically tries `.tsx`, `.ts`, `.jsx`, `.js`, `.mjs`, `.json`, and `.css` extensions, as well as `index` files.

## Rationale

These changes are essential for supporting real-world projects that:
- Have nested TypeScript dependencies
- Use path aliases for cleaner imports
- Import JSON data files
- Have standard frontend dependencies (CSS files) that must be handled during SSR

## Verification

Verified by building and running a production-grade news application with:
- Nested component structures using `@/` aliases
- TypeScript handlers importing JSON data
- Global and component-level CSS imports
- Successful SSR pre-rendering without `.ts` or `.json` errors
