# Driving Status and One-Time Analysis

## Continuously Monitor Driving Status

### Handle Status Values

| Status               | English Label      | Default Priority |
| -------------------- | ------------------ | ---------------: |
| `STOPPED`            | Stopped            |                0 |
| `STEADY_DRIVING`     | Steady driving     |                0 |
| `ACCELERATION`       | Acceleration       |                1 |
| `BRAKING`            | Braking            |                2 |
| `LEFT_TURN`          | Left turn          |                3 |
| `RIGHT_TURN`         | Right turn         |                3 |
| `RAPID_ACCELERATION` | Rapid acceleration |                4 |
| `SUDDEN_BRAKING`     | Sudden braking     |                5 |

Related exports:

- `CAR_RUNNING_STATUSES`: all statuses, suitable for iteration;
- `CAR_RUNNING_LABELS`: Chinese labels corresponding to the statuses;
- `CAR_RUNNING_STATUS_PRIORITY`: status priorities;
- `CarRunningStatus`: the TypeScript union type of the eight statuses.

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
