---
name: toooony-watch-dev
description: Implementation and review guidance for Toooony watch face WebView frontends, covering circular 480 × 480 layouts, direct public t4ony APIs, SDK capability boundaries, static assets, server data, and portable build paths. Use when writing, modifying, fixing, refactoring, optimizing, or reviewing watch face frontend code or build configuration.
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

Use the global `t4ony` object directly for APIs declared by `@ziztechnology/miniprogram-api-typings` when the target device runs Toooony Head 1.8.0 or later. Treat the namespace as unavailable on earlier Head versions and guard optional Runtime paths with a host or capability check.

Use `@ziztechnology/dial-library` for driving status, driving expressions, Sticker carousels, in-car multimedia, browser simulation, normalized return shapes, or compatibility handling. Do not call Runtime-injected objects outside the public `window.t4ony` namespace directly.

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

Build artifacts must be loadable from any subpath or local directory.
