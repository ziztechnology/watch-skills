---
name: toooony-fe-spec
description: Framework-agnostic implementation and review standards for frontend projects, covering source layout, code ownership, component boundaries, and file naming. Use when writing, completing, modifying, fixing, refactoring, or reviewing code for web frontends, H5 applications, admin dashboards, or component libraries, especially when adding pages or components, adjusting directory structures, extracting shared code, or splitting components.
---

# Workflow

1. First inspect the dependency manifest, framework configuration, target files, and existing project conventions.
2. Apply the general standards in this file to all frontend tasks.
3. Follow the conventions of the framework in use and the existing project for framework-specific structures such as pages, routes, layouts, and server-client boundaries.

# Project Structure

Frontend projects must use the Src Layout, with all application source code stored in the `src` directory. The following directory responsibilities are framework-agnostic:

- `src/components`: Store general-purpose components reused across pages and business contexts.
- `src/hooks`: Store Hooks or equivalent composable logic reused across components. Follow the naming conventions of the framework in use.
- `src/utils`: Store general-purpose utility functions that have no side effects and are independent of business logic and frameworks.
- `src/lib`: Store third-party library wrappers, infrastructure adapters, and global instances, such as HTTP clients, logging, monitoring, and storage adapters.
- `src/constants`: Store constants and enums reused across modules. Constants used by only one module should be colocated with that module.
- `src/types`: Store type definitions shared across modules. Define component Props and module-private types close to where they are used.
- `src/assets`: Store static assets that must participate in the build process, such as images, fonts, and style resources.

# Code Ownership and Component Boundaries

Prefer organizing code by business domain. Colocate components, composable logic, utility functions, types, and constants used by only one page, component, or business module with their consumer. Use shared directories for code that has stable responsibilities and is reused by multiple independent business modules.

As a rule, define one primary component per component file and place other components in their own files. Extract parts that require independent state, lifecycle, or composable logic into separate components. Local rendering fragments may be extracted into plain functions when the framework permits.

# Programming Paradigm

- Prefer functional programming (FP) to organize logic, using pure functions, immutable data, function composition, and explicit data flow.
- Prefer functions, closures, Hooks, Composables, and plain objects for reuse and state encapsulation.
- Use `class` only when explicitly required by the target framework, a third-party library, or a platform interface. Restrict `class` to adaptation boundaries and separate business logic into independently testable functions.

# Naming and Entry Files

- Directory names must use lowercase or kebab-case consistently.
- Component files must follow the conventions of the target framework and the existing project. Use PascalCase when no convention exists.
- Plain JavaScript and TypeScript files must use kebab-case, for example `format-date.ts`.
- Framework constructs such as Hooks and Composables must follow the naming conventions of the target framework.
- Directory entry files such as `index.ts` must focus on exporting the public API. Place business implementations in module files that match their responsibilities.
