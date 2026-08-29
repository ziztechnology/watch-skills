---
name: toooony-watch-sdk
description: Integration, testing, and review guidance for Toooony watch face frontend code that uses @ziztechnology/dial-library. Use when implementing, fixing, refactoring, migrating, testing, debugging, or reviewing SDK-based sensors, SSO, Bluetooth metadata, driving features, media features, lifecycle handling, or browser simulation.
---

# Workflow

1. Identify the SDK modules, target Runtime, application type, and project-locked `@ziztechnology/dial-library` version involved in the task.
2. Read only the references relevant to the task. Read every corresponding reference when a task spans multiple modules.
3. Inspect the project's existing wrappers, lifecycle entry points, player instances, and error-handling conventions.
4. Use the package's public exports and follow the declarations in the project-locked version. Import browser testing APIs from `@ziztechnology/dial-library/testing` only in test or local-development entry points.
5. Implement the relevant unavailable, failure, and lifecycle paths described in the selected references, then run the project's applicable checks and tests.

# Reference Routing

- Read [runtime-and-sensors.md](references/runtime-and-sensors.md) when handling installation, Runtime requirements, quick integration, device information, sensor fields, or local simulation.
- Read [sso.md](references/sso.md) when requesting a one-time SSO token, calling an SSO-protected endpoint, handling token absence or failure, or applying token security rules.
- Read [bluetooth-now-playing.md](references/bluetooth-now-playing.md) when reading Bluetooth media-session metadata, playback progress, artwork URIs, lyrics, playback capabilities, or the three snapshot status layers.
- Read [driving-status.md](references/driving-status.md) when handling stable driving status, forward or reverse travel direction, sensor-read timeouts, mount stability, controller parameters, diagnostic events, or single-snapshot sensor analysis.
- Read [driving-expressions.md](references/driving-expressions.md) when handling Runtime media configuration, media URLs, Emoji/image/video/Live Photo/TGS playback, or page lifecycle.
- Read [sticker-carousel.md](references/sticker-carousel.md) when handling random Stickers, prefetching, cover fallback, or carousel lifecycle.
- Read [multimedia.md](references/multimedia.md) when handling in-car buttons, web audio players, playlists, HLS, state synchronization, or custom player integration.
- Read [testing.md](references/testing.md) when writing browser-based SDK tests, installing the simulated Runtime, supplying sensor, SSO, or Bluetooth Now Playing providers, injecting driving expression configuration, dispatching multimedia or Runtime lifecycle events, handling test isolation, or configuring local development and HMR cleanup.

# Implementation Requirements

- Use SDK functionality through the public exports of `@ziztechnology/dial-library`. Use the public `window.t4ony` namespace directly only when its raw contract is the intended interface. Do not call other Runtime-injected methods, objects, or legacy hooks directly.
- Preserve the SDK's selected compatibility path. Propagate or handle failures from a selected public t4ony method without retrying the same operation through an internal legacy Bridge.
- Keep Runtime-dependent APIs in browser execution paths. Use the simulated Runtime or application-owned adapters in standard browsers and local tests.
- Implement matching cleanup for every long-running controller, player, carousel, subscription, and event listener.
