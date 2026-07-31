# Installation and Basic Sensors

## Installation

```bash
npm install @ziztechnology/dial-library
```

You may also use pnpm or yarn.

## Verify the Runtime Environment

- Use Toooony Runtime `v1.5.22` or later.
- Before using driving status sensors, verify that the application is an approved `PACKAGED_H5`.
- Call sensor and in-car multimedia APIs only in Toooony Runtime. Use mock data in standard browsers, Node.js, server-side rendering, and local tests.
- Access capabilities through the public exports of `@ziztechnology/dial-library`. Do not use internal methods injected by the Runtime into `window` directly.

## Read Basic Sensors

### Call `unifiedSensorInfo()`

Call `unifiedSensorInfo()` to read the current device information and handle errors when the Runtime is unavailable:

```ts
import { unifiedSensorInfo } from '@ziztechnology/dial-library';

let info;

try {
  info = await unifiedSensorInfo();
} catch (error) {
  console.error('Unable to read device information. Verify that the page is running in a supported Toooony Runtime.', error);
  throw error;
}

if (info.temperature.available) {
  console.log(`Battery temperature: ${info.temperature.value}${info.temperature.unit}`);
} else {
  console.log(`Temperature unavailable: ${info.temperature.unavailableReason}`);
}

if (info.linearAcceleration.available) {
  const { x, y, z } = info.linearAcceleration.value;
  console.log('Linear acceleration:', { x, y, z }, info.linearAcceleration.unit);
}
```

Check `available` before reading `value` from any sensor field:

```ts
if (info.wifiSsid.available) {
  console.log(info.wifiSsid.value);
}
```

Do not use only `value === null` to determine whether the device supports a capability because data may be unavailable for different reasons.

`motionIntensity` is optional. Use optional chaining when reading it:

```ts
if (info.motionIntensity?.available) {
  console.log(info.motionIntensity.value);
}
```

### Select the Required Fields

| Field                | Content                                             | Unit     |
| -------------------- | --------------------------------------------------- | -------- |
| `gyroscope`          | Gyroscope `{ x, y, z }`                             | `rad/s`  |
| `accelerometer`      | Accelerometer `{ x, y, z }`                         | `m/s²`   |
| `linearAcceleration` | Linear acceleration without gravity `{ x, y, z }` | `m/s²`   |
| `gravity`            | Gravity vector `{ x, y, z }`                        | `m/s²`   |
| `magneticField`      | Magnetic field `{ x, y, z }`                        | `µT`     |
| `orientation`        | Orientation `{ azimuth, pitch, roll }`              | `degree` |
| `motionIntensity`    | Motion intensity; older Runtimes may omit it        | `m/s²`   |
| `temperature`        | Battery temperature                                 | `°C`     |
| `wifiSsid`           | Current Wi-Fi name                                  | None     |
| `batteryCapacity`    | Battery percentage and charge counter               | None     |
| `battery`            | Battery presence, charging state, health, voltage, and more | None |

`capturedAtMs` is the time when the device information was generated. `sampledAtMs` in available data is the sampling time for the corresponding sensor. When data is unavailable, this field may be `null` depending on the sensor type. Check `available` before using it.

`levelPercent` and `chargeCounterUah` in `batteryCapacity.value` may each be `null`. This means the Runtime read the battery capacity object but one of its values is temporarily unavailable. Check each value separately before using it.

### Handle Unavailable Fields

When a field is unavailable, use `unavailableReason` to decide whether to show a message, retry, or fall back:

| Value                  | Meaning                                           |
| ---------------------- | ------------------------------------------------- |
| `SENSOR_MISSING`       | The device does not have the corresponding sensor |
| `NO_SAMPLE`            | The sensor exists but has no sample yet           |
| `PLATFORM_UNAVAILABLE` | The current platform does not provide the feature |
| `PERMISSION_DENIED`    | Read permission was not granted                   |
| `NOT_CONNECTED`        | The device or network is not connected            |

`batteryCapacity` is an exception: when unavailable, its `unavailableReason` may also be `null`. This means the Runtime did not provide a more specific failure reason. Handle it directly as "capacity unavailable."

Battery-related types include `BatteryStatus`, `BatteryPlugged`, and `BatteryHealth`. `BATTERY_HEALTH_LABELS` converts battery health states into Chinese labels.
