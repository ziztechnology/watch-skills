# SDK Testing

## Contents

- [Choose the Narrowest Test Boundary](#choose-the-narrowest-test-boundary)
- [Install the Simulated Runtime](#install-the-simulated-runtime)
- [Test Sensor-Dependent Behavior](#test-sensor-dependent-behavior)
- [Test One-Time SSO Tokens](#test-one-time-sso-tokens)
- [Test Bluetooth Now Playing](#test-bluetooth-now-playing)
- [Test Driving Expression Configuration](#test-driving-expression-configuration)
- [Test In-Car Multimedia](#test-in-car-multimedia)
- [Test Runtime and Browser Lifecycle](#test-runtime-and-browser-lifecycle)
- [Isolate Tests and Clean Up](#isolate-tests-and-clean-up)
- [Verify the Complete Test Matrix](#verify-the-complete-test-matrix)

## Choose the Narrowest Test Boundary

Select the test boundary according to the behavior under test:

- Pass a `sensorProvider` and, when necessary, a controlled `clock` directly to `DrivingStatusController` when testing only driving-state classification, confirmation windows, or provider failures.
- Install the simulated Runtime from `@ziztechnology/dial-library/testing` when testing application code that calls Runtime-dependent SDK APIs such as `unifiedSensorInfo()`, `requestSSOToken()`, or `getBluetoothNowPlaying()`, reads Runtime driving expression configuration, receives in-car multimedia commands, or reacts to Runtime lifecycle events.
- Run final compatibility checks in a supported Toooony Runtime. The simulated Runtime verifies frontend integration but does not verify the target Runtime version, `PACKAGED_H5` approval, device permissions, real sensor timing, WebView media support, or native Bridge behavior.

The testing entry requires a browser `Window`. Run it in a real browser or a test environment that provides the browser APIs required by the feature under test. Calling `installTestingRuntime()` in raw Node.js or during server-side rendering throws an error.

## Install the Simulated Runtime

Import production APIs from the package root and import the testing installer separately as the default export:

```ts
import { DrivingStatusController, unifiedSensorInfo } from '@ziztechnology/dial-library';
import installTestingRuntime, { type TestingRuntimeWindow } from '@ziztechnology/dial-library/testing';

const runtimeWindow: TestingRuntimeWindow = installTestingRuntime();

console.log(await unifiedSensorInfo());

const controller = new DrivingStatusController();
controller.start();
```

Install the simulated Runtime before invoking a Runtime-dependent API or creating an SDK object that connects to a Runtime Bridge. Importing the production package before installation is safe. Continue to use the host browser's `document`, timers, network, media elements, and lifecycle events.

The simulated Runtime does not install a complete public t4ony namespace on the real `window`. Put direct t4ony calls behind an application-owned typed adapter, inject a mock adapter in unit tests, and complete final validation in a supported packaged watch face on the target Runtime.

The default simulated Runtime provides the following state:

- `unifiedSensorInfo()` returns a complete stationary-device snapshot with strictly increasing timestamps.
- `requestSSOToken()` returns `null`.
- `getBluetoothNowPlaying()` returns a complete `available: true`, `success: true`, `status: 'IDLE'`, `active: false` snapshot.
- `readDrivingExpressionsConfig()` returns `null` because no driving expression configuration is installed.
- The in-car multimedia Bridge is available and records lifecycle notifications, the last reported player state, and Runtime logs.

Only one simulated Runtime may be installed in the same JavaScript context. Use isolated browser contexts or workers for concurrent tests, or run tests that share one Window serially. A second installation before `restore()` throws an error.

## Test Sensor-Dependent Behavior

Call `setSensorProvider()` to replace the default stationary sensor source. The provider may return a snapshot synchronously or asynchronously, and it runs for every `unifiedSensorInfo()` call and every default `DrivingStatusController` poll.

```ts
import type { UnifiedSensorInfoPayload } from '@ziztechnology/dial-library';

const snapshots: UnifiedSensorInfoPayload[] = loadRecordedSnapshots();
let snapshotIndex = 0;

runtimeWindow.setSensorProvider(() => {
  const snapshot = snapshots[Math.min(snapshotIndex, snapshots.length - 1)];
  snapshotIndex += 1;

  if (!snapshot) throw new Error('No simulated sensor snapshot is available.');
  return snapshot;
});
```

Keep `capturedAtMs` and the relevant sensor `sampledAtMs` values valid and increasing when testing controller time windows. Reusing one timestamp prevents time-based evidence windows from progressing and may cause the controller to treat data as stale or duplicate.

Use a thrown error or rejected Promise to verify temporary Runtime or sensor failures:

```ts
runtimeWindow.setSensorProvider(async () => {
  throw new Error('Simulated sensor failure');
});
```

Verify both the short-unavailable path that retains the last state and the `unavailableFallbackMs` path that eventually falls back to `STOPPED`. Construct fields with `available: false` and the intended `unavailableReason` when testing field-level fallback UI.

Install initial data in one call when the test does not need to change it later:

```ts
import { OFFICIAL_DRIVING_EXPRESSIONS_CONFIG } from '@ziztechnology/dial-library';

const initialSnapshot = snapshots[0];
if (!initialSnapshot) throw new Error('No initial sensor snapshot is available.');

const runtimeWindow = installTestingRuntime({
  sensorProvider: () => initialSnapshot,
  drivingExpressionsConfig: OFFICIAL_DRIVING_EXPRESSIONS_CONFIG,
});
```

## Test One-Time SSO Tokens

Pass `ssoTokenProvider` during installation when the first application request must receive a token. The Provider receives the Runtime-facing `client_id` and `scope` fields after the public SDK call converts them from `clientID` and `scope`:

```ts
import { requestSSOToken } from '@ziztechnology/dial-library';
import installTestingRuntime, {
  type TestingSSOTokenProvider,
  type TestingSSOTokenRequest,
} from '@ziztechnology/dial-library/testing';

const receivedRequests: TestingSSOTokenRequest[] = [];
const ssoTokenProvider: TestingSSOTokenProvider = (request) => {
  receivedRequests.push(request);

  return {
    sso_token: `${request.client_id}:${request.scope}:testing`,
    expires_in: 300,
    expires_at: Date.now() + 300_000,
    device_id: 123,
    user_id: 456,
    issuer: 'https://api.example.com',
    authorize_url: 'https://api.example.com/api/v1/sso/authorize',
  };
};

const runtimeWindow = installTestingRuntime({ ssoTokenProvider });
const ssoToken = await requestSSOToken({
  clientID: 'finance-watch-face',
  scope: 'finance.read',
});
```

Assert that `receivedRequests` contains `{ client_id: 'finance-watch-face', scope: 'finance.read' }`, and assert the application uses the normalized `ssoToken.token` in the protected request header. Do not use a production token in tests.

Call `setSSOTokenProvider()` to replace the scenario while the test is running:

```ts
runtimeWindow.setSSOTokenProvider(() => null);
console.log(await requestSSOToken()); // null
```

Test these paths independently:

- Omit the Provider to verify the default `null` result.
- Return a valid Runtime-shaped object to verify normalized `SSOToken` fields.
- Return `null` or a malformed object to verify the application's unavailable path.
- Throw synchronously and return a rejected Promise to verify the application's error path.
- Pass empty `clientID` or `scope` values to verify public input validation.

After `restore()`, `setSSOTokenProvider()` and other mutating helpers on that testing object must throw. Create a new simulated Runtime for the next test.

## Test Bluetooth Now Playing

Use `bluetoothNowPlayingProvider` at installation time for a fixed scenario, or call `setBluetoothNowPlayingProvider()` to change the scenario while a test is running. `TestingBluetoothNowPlayingProvider` may return a `BluetoothNowPlayingSnapshot` synchronously or asynchronously.

The default Provider supplies a complete idle snapshot. Reuse it to build a complete playing fixture without duplicating every normalized field:

```ts
import { getBluetoothNowPlaying } from '@ziztechnology/dial-library';
import installTestingRuntime from '@ziztechnology/dial-library/testing';

const runtimeWindow = installTestingRuntime();
const idleSnapshot = await getBluetoothNowPlaying();

runtimeWindow.setBluetoothNowPlayingProvider(() => ({
  ...idleSnapshot,
  available: true,
  success: true,
  status: 'OK',
  active: true,
  title: 'Simulated Track',
  artist: 'Simulated Artist',
  playbackState: 3,
  playbackStateName: 'PLAYING',
  canPause: true,
}));

const playingSnapshot = await getBluetoothNowPlaying();
```

Supply an existing complete fixture during installation when the scenario must be active on the first application read:

```ts
const runtimeWindow = installTestingRuntime({
  bluetoothNowPlayingProvider: async () => playingFixture,
});
```

Test the state branches independently:

- Use `available: false`, `success: false`, and `active: false` for an unavailable DeviceAgent channel.
- Use `available: true`, `success: false`, and `active: false` for a Provider permission or read failure.
- Use `available: true`, `success: true`, and `active: false` for an idle channel with no valid media session.
- Use all three as true for active metadata, progress, artwork, lyrics, and capability UI.

Use a thrown error and a rejected Promise as separate Provider cases. Assert that application error handling receives the same error object:

```ts
runtimeWindow.setBluetoothNowPlayingProvider(() => {
  throw new Error('Simulated synchronous Bridge failure');
});

runtimeWindow.setBluetoothNowPlayingProvider(() => Promise.reject(new Error('Simulated asynchronous Bridge failure')));
```

Restore the simulated Runtime in teardown. After `restore()`, `setBluetoothNowPlayingProvider()` and other mutating helpers on that testing object must throw. Create a new simulated Runtime for the next test instead of reusing the restored object.

## Test Driving Expression Configuration

Set, replace, or clear the Runtime-provided configuration while the page is running:

```ts
import { OFFICIAL_DRIVING_EXPRESSIONS_CONFIG, readDrivingExpressionsConfig } from '@ziztechnology/dial-library';

runtimeWindow.setDrivingExpressionsConfig(OFFICIAL_DRIVING_EXPRESSIONS_CONFIG);
const config = readDrivingExpressionsConfig();

runtimeWindow.clearDrivingExpressionsConfig();
```

`setDrivingExpressionsConfig()` deliberately accepts `unknown` without pre-validating it. Supply malformed configuration to verify that `readDrivingExpressionsConfig()` rejects it by returning `null` and that the application applies its intended fallback. Call `parseDrivingExpressionsConfig()` directly when the test must assert a specific `DrivingExpressionsConfigError`. After `clearDrivingExpressionsConfig()`, `readDrivingExpressionsConfig()` returns `null`.

## Test In-Car Multimedia

Create a `MultimediaPlayer` or `MultimediaController` after installing the simulated Runtime. Dispatch native-style commands through the returned testing object:

```ts
runtimeWindow.dispatchMultimediaCommand('play');
runtimeWindow.dispatchMultimediaCommand('next');
runtimeWindow.dispatchMultimediaCommand('seekToMs', 12_000);

const requestedState = runtimeWindow.requestMultimediaState();
const bridgeSnapshot = runtimeWindow.getMultimediaBridgeSnapshot();
```

Use `play`, `pause`, `toggle`, `next`, `previous`, or `seekToMs`. Pass `positionMs` only with `seekToMs`, and use a non-negative finite value.

`requestMultimediaState()` reads the current state directly from the connected controller when one is available. Otherwise, it dispatches a state-request event and returns a copy of the most recently recorded player state, or `null` when no state has been recorded. If reading or requesting state throws, the simulated Bridge records an error log and the method returns `null`. The recorded state remains available after the controller is destroyed, so do not use a non-null result to infer that a controller is still connected. `getMultimediaBridgeSnapshot()` returns copies of:

- `lifecycle`: `idle`, `ready`, or `destroyed`;
- `lifecycleEvents`: every Ready and Destroyed notification;
- `lastPlayerState`: the last complete player state reported to the Runtime;
- `logs`: the SDK messages reported through the simulated native Bridge.

Assert the `ready` lifecycle after creating the multimedia controller, assert control callback effects after dispatching commands, and assert `destroyed` after calling `destroy()`. Await any asynchronous player or callback work before checking the resulting state.

## Test Runtime and Browser Lifecycle

Use the simulated Runtime helpers to test SDK subscribers that respond to native pause and resume callbacks:

```ts
runtimeWindow.dispatchRuntimePause();
runtimeWindow.dispatchRuntimeResume();
```

Dispatch real browser events on the host Window or `document` to test browser lifecycle paths. Verify `visibilitychange`, `pagehide`, `pageshow`, `freeze`, and `resume` separately when application behavior depends on them.

`dispatchRuntimePause()` and `dispatchRuntimeResume()` simulate native lifecycle delivery. `MultimediaPlayer` and `MultimediaController` do not automatically implement application page pause and resume policy. Test the application's media lifecycle handlers through the browser events or adapter they consume, then assert that the application pauses, reports state when required, and destroys the player only when the page exits.

## Isolate Tests and Clean Up

Destroy every controller, player, and carousel before restoring the real Runtime Window. Unsubscribe listeners before destroying their owner when the API exposes a separate unsubscribe function.

```ts
const cleanup = () => {
  unsubscribe?.();
  controller?.destroy();
  player?.destroy();
  carousel?.destroy();
  runtimeWindow.restore();
};
```

Call cleanup from the test runner's teardown hook even when an assertion fails. `restore()` is idempotent, but mutating or event-dispatch methods on that testing object throw after restoration. Do not assume that `restore()` destroys SDK objects; resource cleanup remains the application's responsibility.

For Vite local development, perform the same cleanup during hot replacement:

```ts
if (import.meta.hot) {
  import.meta.hot.dispose(cleanup);
}
```

Keep the `/testing` import in a dedicated test or local-development entry that production code cannot reach. Do not import it from the production watch face entry or from code executed inside a real Toooony Runtime.

Cover the relevant success, unavailable, failure, lifecycle, and cleanup paths described above.
