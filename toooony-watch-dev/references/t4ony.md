# Public T4ony API

## Contents

- [Configure TypeScript](#configure-typescript)
- [Use the Public Runtime Namespace](#use-the-public-runtime-namespace)
- [Inspect Runtime Capabilities](#inspect-runtime-capabilities)
- [Handle T4ony Errors](#handle-t4ony-errors)
- [Read Runtime, Window, Device, and Battery Information](#read-runtime-window-device-and-battery-information)
- [Control the Display](#control-the-display)
- [Use Files](#use-files)
- [Use JSON Storage](#use-json-storage)
- [Handle Head Lifecycle Events](#handle-head-lifecycle-events)
- [Read and Observe Network Status](#read-and-observe-network-status)
- [Read Basic Sensors](#read-basic-sensors)
- [Read Bluetooth Now Playing](#read-bluetooth-now-playing)
- [Request an SSO Token](#request-an-sso-token)
- [Test Application Code](#test-application-code)

## Configure TypeScript

Install the public type declarations as a development dependency:

```bash
pnpm add --save-dev @ziztechnology/miniprogram-api-typings
```

Use the equivalent development-dependency command when the project uses npm or yarn.

Append the package to the `compilerOptions.types` array in the TypeScript configuration that compiles the watch face. Preserve all existing entries:

```json
{
  "compilerOptions": {
    "types": ["node", "vite/client", "@ziztechnology/miniprogram-api-typings"]
  }
}
```

This reference targets `@ziztechnology/miniprogram-api-typings@0.0.1`. The package declares the global `t4ony` object. Do not import a runtime value from the package. Import public types only when an explicit annotation or application adapter needs them:

```ts
import type {
  NetworkStatus,
  T4ony,
  T4onyError,
} from '@ziztechnology/miniprogram-api-typings';
```

Follow the declarations in the project-locked package version when its API differs from the examples in this reference.

## Use the Public Runtime Namespace

Target Toooony Head 1.8.0 or later when using the public `t4ony` namespace. Earlier Head versions do not provide this public contract; detect an unavailable host and use a supported SDK fallback when the product must still run there.

Call the global `t4ony` object directly in a supported `PACKAGED_H5` watch face:

```ts
const runtimeInfo = await t4ony.getRuntimeInfo();
```

`window.t4ony` refers to the same public namespace. This namespace is available to supported packaged watch faces without an old `customFields` Handler. Do not assume it exists in a standard browser, Node.js, server-side rendering, or an `EXTERNAL_URL_H5` page.

Keep Runtime calls in browser-only code. Check the host before entering an optional Runtime path:

```ts
const hasT4ony =
  typeof window !== 'undefined' && typeof window.t4ony === 'object';

if (!hasT4ony) {
  showRuntimeUnavailable();
} else {
  const runtimeInfo = await t4ony.getRuntimeInfo();
  showRuntimeVersion(runtimeInfo.runtimeVersion);
}
```

Only methods declared on the public `T4ony` interface may be called directly. Use `@ziztechnology/dial-library` for its public abstractions over driving status, driving expressions, Sticker playback, in-car multimedia, and other internal compatibility Bridges. Do not call `window.unifiedSensorInfo`, `window.AndroidCarBridge`, `window.__carBridge`, legacy lifecycle hooks, or other Runtime-injected compatibility objects from application code.

## Inspect Runtime Capabilities

Call `getRuntimeInfo()` to read the API version, Runtime version, platform, profile, and current state of every public API:

```ts
const runtimeInfo = await t4ony.getRuntimeInfo();
const sensorCapability = runtimeInfo.capabilities.getSensorSnapshot;

switch (sensorCapability.state) {
  case 'available':
    renderSensorUI();
    break;
  case 'inactive':
  case 'denied':
  case 'unavailable':
    renderSensorUnavailable(sensorCapability.reason);
    break;
}
```

Treat capability state as a current observation and still handle a rejection from the subsequent API call. The public API does not provide `canIUse`; do not invent or call that method.

## Handle T4ony Errors

Public calls reject with an `Error` that may expose the `T4onyError` fields `api`, `code`, and optional `details`. Use the stable `code` for application behavior:

```ts
import type { T4onyError } from '@ziztechnology/miniprogram-api-typings';

const isT4onyError = (error: unknown): error is T4onyError =>
  error instanceof Error &&
  typeof (error as Partial<T4onyError>).api === 'string' &&
  typeof (error as Partial<T4onyError>).code === 'string';

try {
  await t4ony.accessFile({ directory: 'data', path: 'profile.json' });
} catch (error) {
  if (isT4onyError(error) && error.code === 'NOT_FOUND') {
    showFirstRunState();
  } else {
    throw error;
  }
}
```

Handle permission, availability, stale page, invalid input, quota, I/O, busy, abort, timeout, unsupported, and internal failures through their declared `T4onyErrorCode` values. Do not branch on the human-readable message or expect old `BRIDGE_*` error codes from a public t4ony call.

## Read Runtime, Window, Device, and Battery Information

Use the four information APIs independently or in parallel:

```ts
const [windowInfo, runtimeInfo, deviceInfo, batteryInfo] = await Promise.all([
  t4ony.getWindowInfo(),
  t4ony.getRuntimeInfo(),
  t4ony.getDeviceInfo(),
  t4ony.getBatteryInfo(),
]);

console.log(windowInfo.windowWidth, windowInfo.windowHeight);
console.log(runtimeInfo.apiVersion, runtimeInfo.runtimeVersion);
console.log(deviceInfo.brand, deviceInfo.model, deviceInfo.androidApiLevel);
console.log(batteryInfo.level, batteryInfo.isCharging);
```

Use `getWindowInfo()` values as CSS-pixel measurements. Treat optional device and battery fields as absent when the Runtime does not return them. Do not expect a serial number or vendor-private system version from `getDeviceInfo()`.

## Control the Display

Use `setKeepScreenOn()`, `setScreenBrightness()`, and `getScreenBrightness()` with their exact option shapes:

```ts
await t4ony.setKeepScreenOn({ keepScreenOn: true });
await t4ony.setScreenBrightness({ mode: 'manual', value: 0.65 });

const brightness = await t4ony.getScreenBrightness();
console.log(brightness.mode, brightness.value);

await t4ony.setScreenBrightness({ mode: 'system' });
await t4ony.setKeepScreenOn({ keepScreenOn: false });
```

A manual brightness value must be a finite number from `0` through `1`. Do not include `value` with `mode: 'system'`. Display settings apply to the current page session; set the intended values again after the application intentionally starts a new page session.

## Use Files

The file API exposes three virtual directories:

- Use `data` for durable application files.
- Use `cache` for disposable files that the application can recreate.
- Use `package` to read files shipped with the current watch-face package. It is read-only.

Pass a relative path for ordinary entries. Do not pass an absolute path, backslash, NUL, empty path segment, `.` segment, `..` segment, or trailing slash. Use `path: ''` only with `accessFile()`, `statFile()`, or `readDirectory()` when addressing the root of a virtual directory. Call `getFileSystemInfo()` before selecting chunk sizes or making quota-sensitive writes:

```ts
const fileSystemInfo = await t4ony.getFileSystemInfo();
const dataVolume = fileSystemInfo.volumes.find(
  (volume) => volume.directory === 'data',
);

if (!dataVolume || !dataVolume.writable) {
  throw new Error('The data volume is unavailable.');
}

console.log(dataVolume.remainingBytes);
console.log(fileSystemInfo.limits.maxWriteChunkBytes);
```

### Create, write, append, and read text

Specify `encoding: 'utf8'` for every text read or write:

```ts
await t4ony.createDirectory({
  directory: 'data',
  path: 'settings',
  recursive: true,
});

const writeResult = await t4ony.writeFile({
  directory: 'data',
  path: 'settings/events.log',
  data: 'watch face started\n',
  encoding: 'utf8',
  overwrite: true,
  createParents: true,
});

await t4ony.appendFile({
  directory: 'data',
  path: 'settings/events.log',
  data: 'watch face resumed\n',
  encoding: 'utf8',
});

const textFile = await t4ony.readFile({
  directory: 'data',
  path: 'settings/events.log',
  encoding: 'utf8',
});

console.log(writeResult.bytesWritten, textFile.data, textFile.sizeBytes);
```

### Read bounded binary chunks

Use a real `ArrayBuffer` for binary writes. Read large binary files in bounded chunks and advance by `bytesRead` until `endOfFile` is true:

```ts
const { limits } = await t4ony.getFileSystemInfo();
const chunks: Uint8Array[] = [];
let offset = 0;
let endOfFile = false;

while (!endOfFile) {
  const result = await t4ony.readFile({
    directory: 'package',
    path: 'assets/model.bin',
    encoding: 'binary',
    offset,
    length: limits.maxReadChunkBytes,
  });

  chunks.push(new Uint8Array(result.data));
  offset += result.bytesRead;
  endOfFile = result.endOfFile;

  if (!endOfFile && result.bytesRead === 0) {
    throw new Error('The binary read made no progress.');
  }
}
```

Keep each binary write or append within `maxWriteChunkBytes` and keep the resulting file within `maxFileBytes`.

### Inspect and manage entries

Use the remaining file methods for existence checks, metadata, pagination, copy, move, and removal:

```ts
await t4ony.accessFile({ directory: 'data', path: 'settings/events.log' });

const stat = await t4ony.statFile({
  directory: 'data',
  path: 'settings/events.log',
});

const firstPage = await t4ony.readDirectory({
  directory: 'data',
  path: 'settings',
  limit: 50,
});

const secondPage = firstPage.nextCursor
  ? await t4ony.readDirectory({
      directory: 'data',
      path: 'settings',
      limit: 50,
      cursor: firstPage.nextCursor,
    })
  : null;

await t4ony.copyFile({
  source: { directory: 'data', path: 'settings/events.log' },
  destination: { directory: 'cache', path: 'events-copy.log' },
  overwrite: true,
});

await t4ony.moveFile({
  source: { directory: 'cache', path: 'events-copy.log' },
  destination: { directory: 'cache', path: 'events-archive.log' },
  overwrite: true,
});

await t4ony.removeFile({
  directory: 'cache',
  path: 'events-archive.log',
  ignoreIfNotExists: true,
});

await t4ony.removeDirectory({
  directory: 'cache',
  path: 'temporary',
  recursive: true,
  ignoreIfNotExists: true,
});

console.log(stat.type, firstPage.entries, secondPage?.entries);
```

`readDirectory()` lists direct children only. Follow `nextCursor` until it is absent when the complete directory listing is required.

## Use JSON Storage

Use `t4ony` Storage for data that must survive a packaged watch-face update. The Runtime gives packaged H5 content an exact virtual origin derived from the watch-face identity and resource revision, so an updated package receives a different origin. Native `localStorage` is scoped to that exact origin and the new revision cannot read values stored under the old one. Use `localStorage` only for revision-local data that may be discarded after an update. The Runtime isolates `t4ony` Storage by `miniProgramId` (`faceId` for a packaged watch face), independently of the resource revision, and preserves that namespace when it removes an old package revision or evicts resource cache. The namespace is removed when the watch-face instance is explicitly deleted, all instances are unbound, or application code calls `clearStorage()`.

Storage accepts JSON values. Do not pass functions, `undefined`, symbols, `BigInt`, class instances, or cyclic objects.

```ts
await t4ony.setStorage({
  key: 'preferences',
  data: {
    theme: 'dark',
    complications: ['battery', 'weather'],
  },
});

const preferences = await t4ony.getStorage({ key: 'preferences' });
const storageInfo = await t4ony.getStorageInfo();

console.log(preferences.data, storageInfo.usedBytes, storageInfo.remainingBytes);

await t4ony.removeStorage({ key: 'preferences' });
```

The current Runtime reports a missing `getStorage()` key with `code: 'INVALID_ARGUMENT'` and `details.reason: 'KEY_NOT_FOUND'`. Treat that exact combination as a missing item when it is an expected first-run state. It reports storage quota exhaustion with `code: 'INVALID_ARGUMENT'` and `details.reason: 'QUOTA_EXCEEDED'`. Use `getStorageInfo()` before quota-sensitive writes and still handle the quota error because the available space can change before `setStorage()` completes. Call `clearStorage()` only when the application explicitly intends to remove every key in its own storage namespace:

```ts
await t4ony.clearStorage();
```

## Handle Head Lifecycle Events

Register and remove the same listener function. Registration does not replay an `initial` event that already occurred, so initialize the current UI state separately:

```ts
import type {
  T4onyHeadHideListener,
  T4onyHeadShowListener,
} from '@ziztechnology/miniprogram-api-typings';

const handleHeadShow: T4onyHeadShowListener = (event) => {
  resumeAnimations(event.reason);
};

const handleHeadHide: T4onyHeadHideListener = (event) => {
  pauseAnimations(event.reason);
};

await t4ony.onT4onyHeadShow(handleHeadShow);
await t4ony.onT4onyHeadHide(handleHeadHide);

const disposeHeadLifecycle = async (): Promise<void> => {
  await Promise.all([
    t4ony.offT4onyHeadShow(handleHeadShow),
    t4ony.offT4onyHeadHide(handleHeadHide),
  ]);
};
```

Use `occurredAtElapsedRealtimeMs` for elapsed-time comparisons. Do not compare it with a Unix timestamp.

## Read and Observe Network Status

Call `getNetworkType()` for the initial state, then register one stable listener when live updates are required:

```ts
import type { NetworkStatusListener } from '@ziztechnology/miniprogram-api-typings';

const initialNetworkStatus = await t4ony.getNetworkType();
renderNetworkStatus(initialNetworkStatus);

const handleNetworkStatusChange: NetworkStatusListener = (status) => {
  renderNetworkStatus(status);
};

await t4ony.onNetworkStatusChange(handleNetworkStatusChange);

const disposeNetworkStatus = async (): Promise<void> => {
  await t4ony.offNetworkStatusChange(handleNetworkStatusChange);
};
```

Use `isConnected`, `networkType`, `hasSystemProxy`, and `weakNet` directly. Treat `signalStrengthDbm` as optional.

## Read Basic Sensors

Call `getSensorSnapshot()` for a point-in-time set of basic sensor and device metrics. Check each metric's `available` discriminator before reading `value`:

```ts
const snapshot = await t4ony.getSensorSnapshot();

if (snapshot.accelerometer.available) {
  const { x, y, z } = snapshot.accelerometer.value;
  renderAcceleration({ x, y, z });
} else {
  renderAccelerationUnavailable(snapshot.accelerometer.unavailableReason);
}

if (snapshot.wifiSsid.available) {
  renderWifiName(snapshot.wifiSsid.value);
}
```

Use `startAccelerometer()` and `startGyroscope()` for continuous samples. Keep the returned subscriptions across temporary page hiding: the Runtime releases the underlying sensor resources while the document is frozen and restores the retained logical subscriptions when it resumes. Stop a subscription only when it is no longer needed, such as during component unmounting or permanent teardown. Starting again is required if application code deliberately stops a subscription while hidden:

```ts
const accelerometerSubscription = await t4ony.startAccelerometer({
  rate: 'ui',
  listener(sample) {
    renderAcceleration(sample);
  },
});

const gyroscopeSubscription = await t4ony.startGyroscope({
  rate: 'normal',
  listener(sample) {
    renderAngularVelocity(sample);
  },
});

const stopSensors = async (): Promise<void> => {
  await Promise.all([
    accelerometerSubscription.stop(),
    gyroscopeSubscription.stop(),
  ]);
};
```

`getSensorSnapshot()` does not provide the atomic `drivingMotionFrame` or `drivingSensorContext` used by the driving model. Use `DrivingStatusController`, `calibrateDrivingSensors()`, and other public exports from `@ziztechnology/dial-library` for driving-status features. Do not call `window.unifiedSensorInfo()` or driving session compatibility methods directly.

## Read Bluetooth Now Playing

Call `getBluetoothNowPlaying()` for one raw public t4ony snapshot:

```ts
const nowPlaying = await t4ony.getBluetoothNowPlaying();

if (!nowPlaying.available) {
  showBluetoothUnavailable(nowPlaying.status, nowPlaying.message);
} else if (nowPlaying.status !== 'OK' && nowPlaying.status !== 'IDLE') {
  showBluetoothReadFailure(nowPlaying.status, nowPlaying.message);
} else if (!nowPlaying.active) {
  showNoActiveBluetoothSession();
} else {
  showTrack({
    title: nowPlaying.title,
    artist: nowPlaying.artist,
    positionMs: nowPlaying.positionMs,
    durationMs: nowPlaying.durationMs,
    canPause: nowPlaying.canPause,
  });
}
```

The direct t4ony result uses `available`, `status`, and `active`. It does not expose the SDK-normalized `success`, `displayTitle`, `displaySubtitle`, or `displayDescription` fields. Use `@ziztechnology/dial-library` when the application needs its normalized `BluetoothNowPlayingSnapshot` contract.

This API reads metadata and playback capabilities only. It does not control playback, subscribe to changes, fetch artwork, or estimate progress. Stop application-owned polling while the page is hidden and prevent overlapping reads.

## Request an SSO Token

Call the public API with no options when the deployed watch face already has its binding, or pass the direct t4ony field names `clientId` and `scope`:

```ts
const tokenResult = await t4ony.requestSSOToken({
  clientId: 'finance-watch-face',
  scope: 'finance.read',
});

const response = await fetch(
  new URL('/api/v1/finance/a-stock-app', tokenResult.issuer),
  {
    headers: {
      'X-Toooony-SSO-Token': tokenResult.ssoToken,
    },
  },
);
```

The direct result uses `ssoToken`, `expiresInSeconds`, `expiresAtEpochSeconds`, `deviceId`, `userId`, `issuer`, and `authorizeUrl`. These names differ from the SDK-normalized `token`, `expiresAtMs`, `deviceID`, `userID`, and `authorizeURL` fields.

Request a fresh one-time token immediately before each protected request. Send it only through the required request header. Do not put it in a URL, log, analytics event, persistent cache, browser storage, or component state. Do not reuse the same token for a retry. A public t4ony SSO call does not require the old `requestSSOToken` `customFields` Handler; that Handler applies only to an SDK compatibility fallback for an older Runtime.

## Test Application Code

The typings package provides declarations, not a browser simulator. The SDK testing entry does not install a complete public t4ony object on the real `window`.

Put direct calls behind an application-owned adapter when they require unit tests, and inject a typed mock for that adapter:

```ts
import type { T4ony } from '@ziztechnology/miniprogram-api-typings';

export type BatteryAPI = Pick<T4ony, 'getBatteryInfo'>;

export const runtimeBatteryAPI: BatteryAPI = {
  getBatteryInfo: () => t4ony.getBatteryInfo(),
};
```

Use a plain object that implements `BatteryAPI` in unit tests. Do not replace the production global t4ony object. Complete final validation in a supported packaged watch face on the target Toooony Runtime.
