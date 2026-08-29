# Driving Status and One-Time Analysis

## Contents

- [Continuously Monitor Driving Status](#continuously-monitor-driving-status)
- [Set the Travel Direction](#set-the-travel-direction)
- [Provide Sensor Data Locally](#provide-sensor-data-locally)
- [Read the Current Controller State](#read-the-current-controller-state)
- [Tune Detection Parameters](#tune-detection-parameters)
- [Handle Controller Diagnostics](#handle-controller-diagnostics)
- [Debug a Single Snapshot](#debug-a-single-snapshot)

## Continuously Monitor Driving Status

### Handle Status Values

| Status               | English Label      |
| -------------------- | ------------------ |
| `STOPPED`            | Stopped            |
| `STEADY_DRIVING`     | Steady driving     |
| `ACCELERATION`       | Acceleration       |
| `RAPID_ACCELERATION` | Rapid acceleration |
| `BRAKING`            | Braking            |
| `LEFT_TURN`          | Left turn          |
| `RIGHT_TURN`         | Right turn         |

Related exports:

- `CAR_RUNNING_STATUSES`: all statuses, suitable for iteration;
- `CAR_RUNNING_LABELS`: Chinese labels corresponding to the statuses;
- `CarRunningStatus`: the TypeScript union type of the seven statuses.

For SDK 0.1.0 and later, treat this table as the complete public status set. Remove legacy `SUDDEN_BRAKING` branches when upgrading. Let `DrivingStatusController` resolve concurrent driving evidence; do not import or recreate the removed `CAR_RUNNING_STATUS_PRIORITY` map.

### Continuously Monitor the Stable Status

When the page must display the driving status continuously, create a `DrivingStatusController`, read the current snapshot first, and then subscribe to subsequent changes:

```ts
import {
  CAR_RUNNING_LABELS,
  DRIVING_TRAVEL_DIRECTION,
  DrivingStatusController,
} from '@ziztechnology/dial-library';

const controller = new DrivingStatusController({
  initialStatus: 'STOPPED',
  initialTravelDirection: DRIVING_TRAVEL_DIRECTION.FORWARD,
  pollIntervalMs: 200,
  sensorReadTimeoutMs: 1_000,
  unavailableFallbackMs: 3_000,
});

const unsubscribe = controller.subscribe((event) => {
  console.log(`${CAR_RUNNING_LABELS[event.previousStatus]} -> ${CAR_RUNNING_LABELS[event.status]}`, event.reason);
});

controller.start();

// Pause temporarily
controller.stop();

// Start again when needed
controller.start();

// Release permanently
unsubscribe();
controller.destroy();
```

Set `pollIntervalMs` and `unavailableFallbackMs` as needed. After sensors remain unavailable for `unavailableFallbackMs`, the controller falls back to `STOPPED`.

Set `sensorReadTimeoutMs` to limit one sensor read. A timeout is treated as temporary sensor unavailability and is reported through `onDiagnostic`; it does not throw from `start()`.

Treat `BRAKING` as the only public braking status. Do not add separate application branches for braking strength.

## Set the Travel Direction

Pass an explicit initial direction when the application knows the current gear:

```ts
const controller = new DrivingStatusController({
  initialTravelDirection: DRIVING_TRAVEL_DIRECTION.REVERSE,
});
```

Update the controller whenever the gear changes:

```ts
controller.setTravelDirection(DRIVING_TRAVEL_DIRECTION.FORWARD);
controller.setTravelDirection(DRIVING_TRAVEL_DIRECTION.REVERSE);
```

Pass `null` when the application no longer has an external direction and wants the controller to resume automatic direction detection:

```ts
controller.setTravelDirection(null);
```

The call immediately clears the external override and sets the current direction and source to `null`. The controller may infer a new direction after a sufficient clean longitudinal evidence window; the call itself does not wait for a newly confirmed stop. When an automatically managed direction reaches a confirmed stop, the controller resets it to `null` so the next launch can be inferred again.

Omitting `initialTravelDirection` preserves the compatibility default of `FORWARD`. Pass `initialTravelDirection: null` to start without that default. Prefer explicit `setTravelDirection()` updates whenever vehicle gear data is available.

## Provide Sensor Data Locally

For local development, testing, or recorded road-test playback, return a complete `UnifiedSensorInfoPayload` through `sensorProvider`:

```ts
import { DrivingStatusController, type UnifiedSensorInfoPayload } from '@ziztechnology/dial-library';

const createMockController = (
  recordedSnapshots: UnifiedSensorInfoPayload[],
  stationarySnapshot: UnifiedSensorInfoPayload,
) => {
  const snapshots = [...recordedSnapshots];

  return new DrivingStatusController({
    sensorProvider: async (signal) => {
      if (signal?.aborted) throw new DOMException('The sensor read was aborted.', 'AbortError');
      return snapshots.shift() ?? stationarySnapshot;
    },
  });
};
```

Use increasing `sampledAtMs` values for consecutive samples. Accept the optional `AbortSignal` and pass it to cancellable work performed by a custom provider. Custom providers are also subject to `sensorReadTimeoutMs`.

Use complete road-test samples when testing acceleration, braking, turning, reverse travel, and mount movement. Use stationary mock data only to verify wiring and the stopped status.

## Read the Current Controller State

```ts
const snapshot = controller.getSnapshot();

console.log(snapshot.status); // Currently confirmed status
console.log(snapshot.candidate); // Candidate status being confirmed; may be null
console.log(snapshot.baseStatus); // STOPPED or STEADY_DRIVING
console.log(snapshot.activeTurn); // Currently locked turn; may be null
console.log(snapshot.lifecycle); // stopped, running, or destroyed
console.log(snapshot.controllerId); // Stable ID shared by this controller's diagnostics
console.log(snapshot.travelDirection); // FORWARD, REVERSE, or null
console.log(snapshot.travelDirectionSource); // default, inferred, external, or null
console.log(snapshot.mountStability); // STABLE or UNSTABLE
```

Use `travelDirectionSource` to distinguish an explicit gear update from the compatibility default or automatic detection. Compare `mountStability` with `DRIVING_MOUNT_STABILITY.STABLE` or `DRIVING_MOUNT_STABILITY.UNSTABLE` when the application needs diagnostics or a recalibration indicator; continue to display the controller's public `status` instead of classifying mount movement in application code.

## Tune Detection Parameters

Default parameters are stored in `DEFAULT_DRIVING_STATUS_TUNING`. They usually do not need to be changed. Override only the fields that require adjustment when real-vehicle testing shows that detection is too slow or too sensitive:

```ts
const controller = new DrivingStatusController({
  tuning: {
    turnYawThreshold: 0.1, // rad/s
    turnConfirmationMs: 500, // ms
    longitudinalThreshold: 0.8, // m/s²
    stoppedConfirmationMs: 1_800, // ms
    mountStabilityWindowMs: 800, // ms
    mountStabilityMinimumSamples: 3,
    maximumMountGravityDirectionChangeDegrees: 7, // degrees
    mountStabilityRecoveryMs: 800, // ms
  },
});
```

Override a small number of parameters only after complete road tests show that detection is too slow or too sensitive. Names ending in `Ms` use milliseconds, acceleration thresholds use `m/s²`, angular velocity uses `rad/s`, `maximumMountGravityDirectionChangeDegrees` uses degrees, and other angle thresholds use radians.

Call `resolveDrivingStatusTuning(overrides)` first to validate and generate the complete configuration, then verify the changes with road-test data.

## Handle Controller Diagnostics

`onDiagnostic` is suitable for debugging and real-vehicle tuning:

```ts
const controller = new DrivingStatusController({
  onDiagnostic(event) {
    if (event.type === 'provider_timeout') {
      console.warn(event.controllerId, `Sensor read exceeded ${event.timeoutMs}ms`);
    }

    if (event.type === 'mount_stability_changed') {
      console.warn(event.controllerId, event.stability, event.maximumGravityDirectionChangeDegrees);
    }

    if (event.type === 'travel_direction_changed') {
      console.log(event.controllerId, event.direction, event.source);
    }
  },
});
```

Every diagnostic event includes the controller's stable `controllerId`. Use it to group logs when a page creates more than one controller.

Handle `provider_timeout`, `mount_stability_changed`, and `travel_direction_changed` when the application needs targeted diagnostics. A subscribed status event may use `reason: 'travel_direction_changed'` when changing direction clears a displayed longitudinal action. Keep existing handling for `candidate_confirmed` and `unavailable_fallback` when switching exhaustively on `DrivingStatusChangeReason`.

## Debug a Single Snapshot

Use `classifyDrivingSnapshot()` to inspect one sensor snapshot. Continue using `DrivingStatusController` when the page displays a stable status:

```ts
import { classifyDrivingSnapshot, unifiedSensorInfo } from '@ziztechnology/dial-library';

const analysis = classifyDrivingSnapshot(await unifiedSensorInfo());

if (!analysis) {
  console.log('Required sensor data is missing, invalid, or stale');
} else {
  console.log('Longitudinal acceleration:', analysis.metrics.longitudinal);
  console.log('Lateral acceleration:', analysis.metrics.lateral);
  console.log('Yaw rate:', analysis.metrics.yaw);
  console.log('Instantaneous candidate status:', analysis.candidateStatus);
  console.log('Rejection due to bumps:', analysis.rejectionReason);
}
```

Do not use `candidateStatus` as the page's final status. The page UI must use the stable status returned by `DrivingStatusController`.
