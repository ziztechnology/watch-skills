---
name: toooony-watch-sdk
description: Integration, implementation, testing, and review guidance for the Toooony watch face frontend SDK @ziztechnology/dial-library, covering sensors, one-time SSO tokens, Bluetooth Now Playing snapshots, the SDK 0.1.0+ seven-state driving model, SDK 0.1.2+ travel direction, schema v2 driving expressions and legacy migration, media playback, Sticker carousels, in-car multimedia, lifecycle handling, and browser simulation through /testing. Use when writing, modifying, fixing, refactoring, testing, or reviewing watch face frontend code that uses this SDK, including SSO-authenticated requests, Runtime capability handling, Bluetooth media-session metadata, driving-status migration and tuning, local providers, multimedia commands, cleanup, debugging, and troubleshooting.
---

# Workflow

1. Identify the SDK modules, target Runtime, application type, and project-locked `@ziztechnology/dial-library` version involved in the task.
2. Follow "Reference Routing" and read only the references relevant to the current task. Read every corresponding reference when a task spans multiple modules.
3. Inspect the project's existing wrappers, lifecycle entry points, player instances, and error-handling conventions. Keep the implementation consistent with the existing architecture.
4. Implement functionality using only APIs and types exposed through the package's public exports. Import production APIs from `@ziztechnology/dial-library`, and import browser testing APIs from `@ziztechnology/dial-library/testing` only in test or local-development entry points. When types or behavior vary by version, follow the public type declarations in the project-locked version.
5. Define explicit handling paths for a missing or invalid SSO token, an SSO request failure, an unavailable Runtime, an unavailable DeviceAgent channel, failed snapshot reads, inactive Bluetooth media sessions, insufficient permissions, missing sensor samples, invalid configuration, media loading failures, and page suspension.
6. Unsubscribe and destroy controllers, players, and carousel instances when the page or component unmounts. When a page is temporarily hidden, pause as described in the corresponding reference and resume after the page becomes visible again.
7. Run the project's existing type checks, format checks, and relevant tests. Verify pause, resume, destruction, and failure fallback paths.

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

- Use sensors, SSO, Bluetooth Now Playing, and in-car multimedia capabilities only through `@ziztechnology/dial-library`. Do not call internal methods injected by the Runtime into `window` directly.
- Request a fresh SSO token for each protected request. Handle `null` separately from rejected calls, send the token only through the required request header, and never place it in a URL, log, cache, or component state.
- Treat a missing Runtime or missing `requestSSOToken` method as a `null` SSO result. Preserve rejections from an available Runtime method as failures.
- Enable the `requestSSOToken` Handler in the watch-face template's `customFields` before using SSO. Do not retry a failed HTTP request with the same one-time token.
- Read Bluetooth media sessions with the single-snapshot `getBluetoothNowPlaying()` API. Handle `available`, `success`, and `active` as separate state layers, and keep inactive or failed snapshots as data instead of converting them to `null`.
- Enable Bluetooth Now Playing only for packaged H5 watch faces by setting `getBluetoothNowPlaying`, `bluetoothNowPlaying`, `nowPlaying`, or `musicNowPlaying` to true in `customFields`. Prefer `getBluetoothNowPlaying`. Treat the complete Bridge as unavailable to external URL H5 pages.
- Driving status sensors require Toooony Runtime `v1.5.22` or later and are available only to approved `PACKAGED_H5` applications.
- For projects using SDK 0.1.0 or later, implement the seven public driving statuses and schema v2 driving expression configuration. Treat schema v1 as migration input only, and consume the normalized v2 result.
- Sensors, SSO, Bluetooth Now Playing, and in-car multimedia depend on Toooony Runtime. Use the simulated Runtime or isolate Runtime calls in standard browsers and local tests; keep browser-only Runtime APIs out of Node.js and server-side rendering execution paths.
- Import `@ziztechnology/dial-library/testing` only from browser-based test or local-development entry points. Install its simulated Runtime before invoking Runtime-dependent APIs or creating Runtime-dependent SDK objects, and restore it after destroying those objects.
- Check `available` before reading a sensor field's `value`. Distinguish failure causes through `unavailableReason`.
- Implement matching cleanup logic for every long-running controller, player, carousel, subscription, and event listener.
