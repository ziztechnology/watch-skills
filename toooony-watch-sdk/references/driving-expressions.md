# Driving Expressions and Player

## Contents

- [Read and Validate the Driving Expression Configuration](#read-and-validate-the-driving-expression-configuration)
- [Validate Media URLs](#validate-media-urls)
- [Create a Driving Expression Player](#create-a-driving-expression-player)
- [Manage the Page Lifecycle](#manage-the-page-lifecycle)

## Read and Validate the Driving Expression Configuration

The Runtime places the configuration in `__TOOOONY_DRIVING_EXPRESSIONS__` on the page. Under normal circumstances, read it directly:

```ts
import { readDrivingExpressionsConfig } from '@ziztechnology/dial-library';

const config = readDrivingExpressionsConfig();

if (!config) {
  console.log('The Runtime did not provide a complete, valid driving expression configuration');
}
```

When the configuration is missing or invalid, the function returns `null`. The SDK does not silently add default media.

### Write a Current Configuration

`CompleteDrivingExpressionsConfig` uses `schemaVersion: 2`. Its `states` object must contain exactly the seven current statuses. Each status may use one of the following media types:

```ts
type StructuredDrivingExpressionMedia =
  | { kind: 'emoji'; text: string }
  | { kind: 'image'; mediaAssetId?: number; url: string; mimeType?: string }
  | { kind: 'video'; mediaAssetId?: number; url: string; coverUrl?: string; mimeType?: string }
  | { kind: 'live_photo'; mediaAssetId?: number; coverUrl: string; motionUrl: string }
  | { kind: 'tgs'; mediaAssetId?: number; url: string; coverUrl?: string };
```

Define media according to these requirements:

- `emoji.text` must contain exactly one visible character or Emoji.
- If present, `mediaAssetId` must be a safe integer greater than 0.
- Do not include conflicting or deprecated fields in media. For example, an image cannot have `coverUrl`, and a Live Photo cannot have `url`.
- Every media URL must pass the public HTTPS URL validation described below.

Minimal example:

```ts
const rawConfig = {
  schemaVersion: 2,
  states: {
    STOPPED: { kind: 'emoji', text: '😐' },
    STEADY_DRIVING: { kind: 'emoji', text: '😊' },
    ACCELERATION: { kind: 'emoji', text: '😄' },
    RAPID_ACCELERATION: { kind: 'emoji', text: '😲' },
    BRAKING: { kind: 'emoji', text: '😣' },
    LEFT_TURN: { kind: 'emoji', text: '🤨' },
    RIGHT_TURN: { kind: 'emoji', text: '🤨' },
  },
};
```

### Migrate a Legacy Configuration

Use schema v1 only as migration input. `parseDrivingExpressionsConfig()` accepts a v1 configuration only when it contains all eight legacy statuses. It returns a complete schema v2 configuration, preserves the legacy `BRAKING` media, and discards the legacy `SUDDEN_BRAKING` media.

Reject incomplete v1 configurations. Reject schema v2 configurations that contain `SUDDEN_BRAKING` or any other unknown status. Always create, persist, and send schema v2 configurations after migration.

### Select a Configuration API

| API                                     | Result                                                        |
| --------------------------------------- | ------------------------------------------------------------- |
| `parseDrivingExpressionsConfig(raw)`    | Throws `DrivingExpressionsConfigError` for invalid configuration |
| `tryParseDrivingExpressionsConfig(raw)` | Returns `null` for invalid configuration                      |
| `readDrivingExpressionsConfig()`        | Reads the Runtime configuration; returns `null` if missing or invalid |
| `waitForDrivingExpressionsConfig()`     | Waits for a Runtime configuration; returns `null` on timeout  |

After catching `DrivingExpressionsConfigError`, use `error.code` to distinguish an incorrect schema, a missing status, invalid media fields, and an unsafe URL. For user-provided configuration, record `code` and `message` to identify the incorrect entry.

The Runtime usually provides the configuration before page scripts run. Wait asynchronously only when the configuration may arrive late:

```ts
import { waitForDrivingExpressionsConfig } from '@ziztechnology/dial-library';

const abortController = new AbortController();
window.addEventListener('pagehide', () => abortController.abort(), { once: true });

const config = await waitForDrivingExpressionsConfig({
  timeoutMs: 3_000,
  pollIntervalMs: 100,
  signal: abortController.signal,
}).catch((error: unknown) => {
  if (error instanceof Error && error.name === 'AbortError') return null;
  throw error;
});
```

On cancellation, the Promise rejects with `AbortError`. The example above catches that error and converts the cancellation result to `null`, preventing an unhandled Promise rejection when the page is left.

Legacy v0 configurations may specify image URLs directly instead of structured media objects. Enable compatibility only while migrating that bare-URL format:

```ts
const config = parseDrivingExpressionsConfig(rawConfig, {
  allowLegacyImageUrls: true,
});
```

`resolveDrivingExpression()` preserves the old partial-config behavior by falling back to `STEADY_DRIVING` media when a requested status is missing. Do not rely on that fallback for newly parsed configurations, which are complete. `resolveDrivingExpressionStrict()` does not automatically use other media; it throws an error when a status is missing and no fallback media is specified. The player uses the latter, strict rule.

`parseDrivingExpressionsConfig()`, `tryParseDrivingExpressionsConfig()`, and `readDrivingExpressionsConfig(raw)` all accept either a parsed object or a JSON string containing that object.

## Validate Media URLs

URLs used for images, videos, Live Photos, and TGS must be long-lived, public HTTPS URLs.

```ts
import { isPublicDrivingMediaUrl } from '@ziztechnology/dial-library';

console.log(isPublicDrivingMediaUrl('https://cdn.example.com/expression.png')); // true
console.log(isPublicDrivingMediaUrl('http://cdn.example.com/expression.png')); // false
console.log(isPublicDrivingMediaUrl('https://localhost/expression.png')); // false
```

The SDK rejects:

- Non-HTTPS URLs, including `http:`, `data:`, and `blob:`.
- URLs containing a username or password.
- localhost, private-network IP addresses, and reserved addresses.
- URLs containing common temporary signature parameters.

Frontend URL validation can only prevent obvious errors. The server must still verify media ownership against `mediaAssetId` and the user identity, then regenerate a trusted URL.

## Create a Driving Expression Player

### Connect the Status Controller to the Player

```html
<div id="expression"></div>

<style>
  #expression {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }
</style>
```

```ts
import {
  createDrivingExpressionPlayer,
  DrivingStatusController,
  readDrivingExpressionsConfig,
} from '@ziztechnology/dial-library';

const container = document.querySelector<HTMLElement>('#expression');
if (!container) throw new Error('Could not find #expression');

const config = readDrivingExpressionsConfig();
if (!config) throw new Error('The Runtime did not provide a valid driving expression configuration');

const player = createDrivingExpressionPlayer(container, { config });
const controller = new DrivingStatusController();

await player.show(controller.getSnapshot().status);

const unsubscribe = controller.subscribe((event) => {
  void player.show(event.status).catch((error) => {
    console.error('Failed to switch the driving expression.', error);
  });
});

controller.start();

const destroy = () => {
  unsubscribe();
  controller.destroy();
  player.destroy();
};

window.addEventListener('pagehide', (event) => {
  if (!event.persisted) destroy();
});
```

The SDK includes built-in TGS playback support. Do not install `lottie-web` separately.

### Update Configuration and Resolve Cached Media

Call `updateConfig()` when the Runtime or application replaces the complete configuration. It reloads the current status even when that status has not changed:

```ts
const player = createDrivingExpressionPlayer(container, {
  config,
  preserveCurrentMediaWhileLoading: true,
  resolveMediaUrl(url, media, source) {
    return mediaCache.resolve(url, media, source) ?? url;
  },
});

await player.show('STOPPED');
await player.updateConfig(nextConfig);
```

The original configuration URLs must remain valid public HTTPS URLs. A trusted `resolveMediaUrl` callback may return a Blob or same-origin cache URL for the `primary`, `cover`, or `motion` source. If the resolved URL fails, the player reports `resolved_media_load_failed` and retries the original URL once. Use `mediaAssetId`, `originalUrl`, and `source` from that diagnostic to invalidate a bad cache entry. Set `preserveCurrentMediaWhileLoading: true` to keep the committed image, video, Live Photo, or TGS visible until its replacement is ready.

### TGS Media Requirements

Meet the following requirements when using TGS media:

- Configure the media server with CORS responses that permit access from the page origin.
- Return `application/gzip`, `application/json`, `application/vnd.telegram.tgsticker`, `application/x-gzip`, or `application/x-tgsticker`.
- By default, limit compressed files to 2 MiB, JSON to 8 MiB, and layers to 500.
- Set `loadTimeoutMs`, `maxTgsCompressedBytes`, `maxTgsJsonBytes`, or `maxTgsLayers` to adjust limits. Pass `fetch` to customize network requests.

The TGS loader also requires a top-level `layers` array. Optional `w` and `h` values must be finite numbers from 1 through 4096. A numeric `fr` must be greater than 0 and no greater than 120. When `ip` and `op` are both numeric, require `op > ip` and a span no greater than 30,000 frames.

Give the player a dedicated `container` with an explicit width and height so images, videos, and animations can use a `contain` or `cover` layout.

The player provides `show()`, `updateConfig()`, `pause()`, `resume()`, `destroy()`, and `getSnapshot()`. The `phase` returned by `getSnapshot()` may be `idle`, `loading`, `ready`, `fallback`, or `error`.

### Use Fallback Media When Media Fails

Official Emoji fallback media is not enabled automatically. Pass it explicitly:

```ts
import { OFFICIAL_DRIVING_EXPRESSION_FALLBACKS, createDrivingExpressionPlayer } from '@ziztechnology/dial-library';

const player = createDrivingExpressionPlayer(container, {
  config,
  fallbacks: OFFICIAL_DRIVING_EXPRESSION_FALLBACKS,
  onDiagnostic(event) {
    if (event.type === 'media_fallback_applied') {
      console.warn(`${event.status} is using ${event.kind} fallback media`);
    }
  },
});
```

You may also provide custom Emoji or fallback images for only some statuses:

```ts
const player = createDrivingExpressionPlayer(container, {
  config,
  fallbacks: {
    STOPPED: { kind: 'emoji', text: '😐' },
    STEADY_DRIVING: { kind: 'image', url: 'https://cdn.example.com/steady.png' },
  },
});
```

Only Emoji and images can be used as fallback media. On network failure, load timeout, decode failure, or invalid configuration, `onDiagnostic` receives `media_load_failed`. If fallback media is selected, it also receives `media_fallback_applied`.

Prefer Emoji when deterministic fallback behavior is required. When using a fallback image, verify the URL and resource availability in advance.

The default `loadTimeoutMs` is 5 seconds per timed attempt or stage, not for the complete operation. TGS network loading and Lottie readiness each receive a separate timeout, while decompression, JSON parsing, and the dynamic Lottie import are outside those timers. A resolved media URL and its fallback to the original URL also receive separate attempts. The player retries the status 15 seconds after a failure by default. Change these values with `loadTimeoutMs` and `retryBackoffMs`. Set image and video display behavior with `fit: 'contain' | 'cover'`.

## Manage the Page Lifecycle

By default, `DrivingStatusController` and the driving expression player automatically pause and resume for the following events:

- The page becomes hidden or visible again.
- `pagehide`, `pageshow`, `freeze`, and `resume`.
- Runtime sensor pause and resume notifications.

Most watch faces therefore do not need to listen to `visibilitychange` themselves. If the page already has centralized lifecycle management, disable automatic management and call the corresponding methods manually:

```ts
const controller = new DrivingStatusController({
  managePageLifecycle: false,
});

const player = createDrivingExpressionPlayer(container, {
  config,
  managePageLifecycle: false,
});
```

After disabling automatic management, the page must call `controller.start()` / `controller.stop()` and `player.resume()` / `player.pause()` itself. Call `destroy()` on both when the page is permanently destroyed.
