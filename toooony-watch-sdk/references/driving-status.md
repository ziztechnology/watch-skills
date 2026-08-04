# Driving Status and One-Time Analysis

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

For SDK 0.1.0 and later, treat this table as the complete public status set. Remove legacy `SUDDEN_BRAKING` branches and imports of the removed `CAR_RUNNING_STATUS_PRIORITY` export when upgrading.

### Apply Status Selection Rules

Let `DrivingStatusController` resolve simultaneous evidence in this order:

1. Strong longitudinal action: `RAPID_ACCELERATION` or strong braking evidence.
2. `LEFT_TURN` or `RIGHT_TURN`.
3. Normal longitudinal action: `ACCELERATION` or normal braking evidence.
4. Base status: `STOPPED` or `STEADY_DRIVING`.

Both normal and strong braking evidence produce the public `BRAKING` status. Do not infer braking strength from `event.status`, a controller snapshot, or the driving expression player. Use diagnostic events when detector-level evidence is required for tuning or troubleshooting.

### Continuously Monitor the Stable Status

When the page must display the driving status continuously, create a `DrivingStatusController`, read the current snapshot first, and then subscribe to subsequent changes:

```ts
import { CAR_RUNNING_LABELS, DrivingStatusController } from '@ziztechnology/dial-library';

const controller = new DrivingStatusController({
  initialStatus: 'STOPPED',
  pollIntervalMs: 200,
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

### Provide Sensor Data Locally

For local development, testing, or recorded road-test playback, return a complete `UnifiedSensorInfoPayload` through `sensorProvider`:

```ts
import { DrivingStatusController, type UnifiedSensorInfoPayload } from '@ziztechnology/dial-library';

const createMockController = (
  recordedSnapshots: UnifiedSensorInfoPayload[],
  stationarySnapshot: UnifiedSensorInfoPayload,
) => {
  const snapshots = [...recordedSnapshots];

  return new DrivingStatusController({
    sensorProvider: async () => snapshots.shift() ?? stationarySnapshot,
  });
};
```

Use increasing `sampledAtMs` values for consecutive samples. Use complete road-test samples when testing acceleration, braking, and turning. Use stationary mock data only to verify wiring and the stopped status.

### Read the Current Controller State

```ts
const snapshot = controller.getSnapshot();

console.log(snapshot.status); // Currently confirmed status
console.log(snapshot.candidate); // Candidate status being confirmed; may be null
console.log(snapshot.baseStatus); // STOPPED or STEADY_DRIVING
console.log(snapshot.activeTurn); // Currently locked turn; may be null
console.log(snapshot.lifecycle); // stopped, running, or destroyed
```

### Tune Detection Parameters

Default parameters are stored in `DEFAULT_DRIVING_STATUS_TUNING`. They usually do not need to be changed. Override only the fields that require adjustment when real-vehicle testing shows that detection is too slow or too sensitive:

```ts
const controller = new DrivingStatusController({
  tuning: {
    turnYawThreshold: 0.1, // rad/s
    turnConfirmationMs: 500, // ms
    longitudinalThreshold: 0.8, // m/s²
    stoppedConfirmationMs: 1_800, // ms
  },
});
```

Override a small number of parameters only after complete road tests show that detection is too slow or too sensitive. Call `resolveDrivingStatusTuning(overrides)` first to validate and generate the complete configuration, then verify the changes with road-test data.

### Diagnose the Detection Process

`onDiagnostic` is suitable for debugging and real-vehicle tuning:

```ts
const controller = new DrivingStatusController({
  onDiagnostic(event) {
    if (event.type === 'detector_transition') {
      console.log(event.detector, event.phase, event.status, event.statistics);
    }

    if (event.type === 'fallback_to_stopped') {
      console.warn(`Sensors have remained unavailable for ${event.unavailableForMs}ms`);
    }
  },
});
```

Record `onDiagnostic` events during tuning and troubleshooting. Focus on invalid samples, candidate status changes, and `fallback_to_stopped`.

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
