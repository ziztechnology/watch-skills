# Bluetooth Now Playing

## Contents

- [Meet the Runtime Capability Requirements](#meet-the-runtime-capability-requirements)
- [Read a Single Snapshot](#read-a-single-snapshot)
- [Handle the Three State Layers](#handle-the-three-state-layers)
- [Use Media and Playback Fields](#use-media-and-playback-fields)
- [Select Artwork and Lyrics](#select-artwork-and-lyrics)
- [Handle Errors and Refreshing](#handle-errors-and-refreshing)

## Meet the Runtime Capability Requirements

Use Bluetooth Now Playing only in a packaged H5 watch face loaded by Toooony Runtime. SDK 0.2.0 first calls the public `t4ony.getBluetoothNowPlaying()` method when it exists. The standard t4ony path does not require an old Handler.

When the application must support an older Runtime through the SDK compatibility fallback, enable one of these `customFields` keys with a true value:

- `getBluetoothNowPlaying` (preferred)
- `bluetoothNowPlaying`
- `nowPlaying`
- `musicNowPlaying`

External URL H5 pages do not receive the complete capability even if they declare one of these keys. Keep an explicit unavailable or alternative UI for unsupported page types and older Runtime versions.

Use only the public SDK API for the normalized contract described in this reference. The public `window.t4ony` namespace may be used directly when the application intentionally wants the raw public t4ony contract. Runtime compatibility methods such as `window.getBluetoothNowPlaying()`, `window.getNowPlaying()`, and `window.flutter_inappwebview.callHandler()` remain internal surfaces for old watch faces and must not be called directly.

## Read a Single Snapshot

Import `getBluetoothNowPlaying()` and the snapshot type from the package root:

```ts
import { getBluetoothNowPlaying, type BluetoothNowPlayingSnapshot } from '@ziztechnology/dial-library';

const snapshot: BluetoothNowPlayingSnapshot = await getBluetoothNowPlaying();
```

The call returns one point-in-time snapshot. It does not subscribe, poll, control playback, resolve artwork, or estimate progress after the response time.

On the standard path, the SDK maps t4ony statuses `OK` and `IDLE` to `success: true`, maps the remaining statuses to `success: false`, and supplies empty `displayTitle`, `displaySubtitle`, and `displayDescription` compatibility fields. It selects the legacy Bridge only when the standard method is missing. A rejection or invalid response from an existing standard method does not fall back to the legacy Bridge.

## Handle the Three State Layers

Branch in this order:

```ts
const snapshot = await getBluetoothNowPlaying();

if (!snapshot.available) {
  showChannelUnavailable(snapshot.message);
} else if (!snapshot.success) {
  showReadFailure(snapshot.status, snapshot.message);
} else if (!snapshot.active) {
  showNoActiveSession();
} else {
  showNowPlaying(snapshot);
}
```

Each layer has a separate meaning:

- `available` reports whether the Runtime can reach the DeviceAgent Bluetooth Now Playing channel. It is the discriminant for `BluetoothNowPlayingSnapshot`.
- `success` reports whether this Provider read succeeded while the channel was available.
- `active` reports whether the successful read found a valid Bluetooth media session.

Keep every branch as a snapshot. Do not convert `available: false`, `success: false`, or `active: false` to `null`, and do not infer one layer from another. An idle result commonly has `available: true`, `success: true`, `status: 'IDLE'`, and `active: false`.

## Use Media and Playback Fields

The normalized snapshot always exposes these known field groups:

- Availability and result: `available`, `source`, `success`, `status`, `message`, and `active`.
- Media metadata: `packageName`, `title`, `artist`, `album`, `genre`, `mediaId`, `displayTitle`, `displaySubtitle`, `displayDescription`, `lyrics`, and `lyricsSupported`.
- Playback: `durationMs`, `positionMs`, `bufferedPositionMs`, `playbackState`, `playbackStateName`, `actions`, `canPlay`, `canPause`, `canNext`, `canPrevious`, and `canSeek`.
- Artwork: `artUri`, `albumArtUri`, `displayIconUri`, and `hasEmbeddedArt`.
- Track and update time: `trackNumber`, `numTracks`, `updatedAtMs`, and `updatedElapsedRealtimeMs`.

Treat text as potentially empty and numeric positions or durations as potentially zero. Use the five `can*` fields for UI capability decisions instead of decoding the numeric `actions` bitmask. Preserve additional JSON fields when an application adapter copies or stores the snapshot because DeviceAgent may extend its response.

## Select Artwork and Lyrics

Try non-empty artwork URIs in this order:

1. `artUri`
2. `albumArtUri`
3. `displayIconUri`

Keep a no-artwork placeholder. The SDK returns raw strings and does not fetch, proxy, or decode them. Native schemes such as `content:` and `android.resource:` may not be directly displayable by an H5 page. `hasEmbeddedArt` indicates metadata evidence only; it does not guarantee browser access to an image.

Show lyrics only when `lyricsSupported` is true and `lyrics.trim()` is non-empty. A true capability flag does not guarantee that the current track includes lyrics.

## Handle Errors and Refreshing

Use `try`/`catch` around the call. Missing methods on both paths or a malformed known response field reject with `TypeError`. A selected Runtime method's rejection propagates unchanged. Preserve public t4ony `api`, `code`, and `details` fields or legacy `handler` and `error` fields when reporting diagnostics. Treat the three false state branches as normal results, not exceptions.

Choose refresh timing in application code. Stop periodic reads while the page is hidden or suspended, prevent overlapping requests, and restart only when the page becomes visible and the UI still needs live metadata. Do not add polling or subscriptions inside an SDK wrapper around `getBluetoothNowPlaying()`.
