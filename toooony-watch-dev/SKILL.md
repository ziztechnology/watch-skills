---
name: toooony-watch-dev
description: Implementation and review standards for watch face WebView projects, covering the circular visible area, the HTML5 runtime environment, vw/vh size conversion based on 480 × 480 designs, direct public t4ony APIs typed by @ziztechnology/miniprogram-api-typings, SDK-only capabilities through @ziztechnology/dial-library, the SDK seven-state driving model and schema v2 driving expression configuration, static assets, server data, and relative build paths. Use when writing, completing, modifying, fixing, refactoring, optimizing, or reviewing watch face frontend pages, styles, asset references, network requests, driving-status integrations, device capability integrations, and build configurations.
---

# Interface and Styles

Implement the watch face as an HTML5 page running in Toooony WebView.

The watch face has a circular visible area. Design and implement the page according to the circular boundary: the background may cover the entire 480 × 480 canvas; text, icons, buttons, and other essential information must remain inside the circular visible area to avoid clipping.

Write styles from a 480 × 480 design and express dimensions with `vw` and `vh`:

- Use `vw` for horizontal dimensions. `1vw` corresponds to `4.8px` in the design.
- Use `vh` for vertical dimensions. `1vh` corresponds to `4.8px` in the design.
- Divide a pixel value from the design by `4.8` to obtain the corresponding `vw` or `vh` value. For example, write a `48px` width as `10vw` and a `96px` height as `20vh`.

# Reference Routing

Read [t4ony.md](references/t4ony.md) when using the public `t4ony` namespace for Runtime, window, device, battery, display, file, storage, Head lifecycle, network, basic sensor, Bluetooth Now Playing, or SSO capabilities. Read the complete reference when a task spans multiple capability groups.

# Access Runtime and Device Capabilities

Use the global `t4ony` object directly for APIs declared by `@ziztechnology/miniprogram-api-typings` when the target device runs Toooony Head 1.8.0 or later. Treat the namespace as unavailable on earlier Head versions and keep any required fallback behind an explicit host or capability check. The public `t4ony` namespace is not an internal Runtime compatibility Bridge.

Install the typings as a development dependency with the project's existing package manager:

```bash
# npm
npm install --save-dev @ziztechnology/miniprogram-api-typings

# pnpm
pnpm add --save-dev @ziztechnology/miniprogram-api-typings

# yarn
yarn add --dev @ziztechnology/miniprogram-api-typings
```

Append the package to the `types` array in the TypeScript configuration used by the watch face. Preserve every existing entry:

```json
{
  "compilerOptions": {
    "types": ["vite/client", "@ziztechnology/miniprogram-api-typings"]
  }
}
```

Use `@ziztechnology/dial-library` for driving status, driving expressions, Sticker carousels, in-car multimedia, normalized SDK return shapes, browser SDK simulation, and capabilities that the SDK intentionally implements over internal compatibility Bridges. Do not call any Runtime-injected `window.xxx` object outside the public `window.t4ony` namespace directly.

Install the SDK when the selected capability requires its public abstraction:

```bash
# npm
npm install @ziztechnology/dial-library

# pnpm
pnpm add @ziztechnology/dial-library

# yarn
yarn add @ziztechnology/dial-library
```

# Implement Driving Status

Inspect the project-locked SDK version before changing a driving-status integration. Apply the following rules to `@ziztechnology/dial-library` 0.1.0 and later:

- Treat `STOPPED`, `STEADY_DRIVING`, `ACCELERATION`, `RAPID_ACCELERATION`, `BRAKING`, `LEFT_TURN`, and `RIGHT_TURN` as the complete public status set. Derive iteration and labels from `CAR_RUNNING_STATUSES` and `CAR_RUNNING_LABELS`.
- Handle both normal and strong braking as `BRAKING`. Do not add a `SUDDEN_BRAKING` branch to current application code.
- Let `DrivingStatusController` resolve concurrent evidence. Its selection order is strong longitudinal action, turn, normal longitudinal action, then base status. Do not import or recreate the removed `CAR_RUNNING_STATUS_PRIORITY` map.
- Create driving expression configurations with `schemaVersion: 2` and exactly the seven current statuses. Use schema v1 only as input when migrating a complete legacy eight-status configuration through `parseDrivingExpressionsConfig()`; persist and send the normalized v2 result.

# Static Assets

Import static assets with `import` instead of using absolute HTML paths directly. For example:

```tsx
import dialBackground from "./assets/dial-background.png";

export const Foo = () => {
  return <img src={dialBackground} alt="Watch face background" />;
};
```

# Server Data

Server data requests must use SWR with cached-data fallback, error retries, and revalidation on reconnection so data remains available and reloads correctly on weak or intermittent networks.

# Build and Release

## Relative Paths

Build artifacts must be loadable from any subpath or local directory.
