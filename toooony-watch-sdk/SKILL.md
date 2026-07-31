---
name: toooony-watch-sdk
description: Integration, implementation, and review guidance for the Toooony watch face frontend SDK @ziztechnology/dial-library, covering basic sensors, driving status detection and one-time analysis, driving expression configuration and playback, media URLs, page lifecycle, Sticker carousels, and in-car multimedia controls. Use when writing, completing, modifying, fixing, refactoring, testing, or reviewing watch face frontend code that uses this SDK, including Runtime Bridge compatibility, local sensor simulation, player resource cleanup, debugging, and troubleshooting.
---

# Workflow

1. Identify the SDK modules, target Runtime, application type, and project-locked `@ziztechnology/dial-library` version involved in the task.
2. Follow "Reference Routing" and read only the references relevant to the current task. Read every corresponding reference when a task spans multiple modules.
3. Inspect the project's existing wrappers, lifecycle entry points, player instances, and error-handling conventions. Keep the implementation consistent with the existing architecture.
4. Implement functionality using only APIs and types publicly exported from the package entry point. When types or behavior vary by version, follow the public type declarations in the project-locked version.
5. Define explicit handling paths for an unavailable Runtime, insufficient permissions, missing sensor samples, invalid configuration, media loading failures, and page suspension.
6. Unsubscribe and destroy controllers, players, and carousel instances when the page or component unmounts. When a page is temporarily hidden, pause as described in the corresponding reference and resume after the page becomes visible again.
7. Run the project's existing type checks, format checks, and relevant tests. Verify pause, resume, destruction, and failure fallback paths.

# Reference Routing

- Read [runtime-and-sensors.md](references/runtime-and-sensors.md) when handling installation, Runtime requirements, quick integration, device information, sensor fields, or local simulation.
- Read [driving-status.md](references/driving-status.md) when handling stable driving status, controller parameters, diagnostic events, or single-snapshot sensor analysis.
- Read [driving-expressions.md](references/driving-expressions.md) when handling Runtime media configuration, media URLs, Emoji/image/video/Live Photo/TGS playback, or page lifecycle.
- Read [sticker-carousel.md](references/sticker-carousel.md) when handling random Stickers, prefetching, cover fallback, or carousel lifecycle.
- Read [multimedia.md](references/multimedia.md) when handling in-car buttons, web audio players, playlists, HLS, state synchronization, or custom player integration.

# Implementation Requirements

- Use sensor and in-car multimedia capabilities only through `@ziztechnology/dial-library`. Do not call internal methods injected by the Runtime into `window` directly.
- Driving status sensors require Toooony Runtime `v1.5.22` or later and are available only to approved `PACKAGED_H5` applications.
- Sensors and in-car multimedia depend on Toooony Runtime. Use mock data or isolate Runtime calls in standard browsers, local tests, Node.js, and server-side rendering environments.
- Check `available` before reading a sensor field's `value`. Distinguish failure causes through `unavailableReason`.
- Implement matching cleanup logic for every long-running controller, player, carousel, subscription, and event listener.
