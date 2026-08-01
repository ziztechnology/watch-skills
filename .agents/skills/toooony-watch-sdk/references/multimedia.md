# In-Car Multimedia

## Create a Complete Audio Player

`MultimediaPlayer` is the recommended framework-agnostic audio player. The application provides an `HTMLAudioElement` and a playlist. The SDK manages playback, track switching, HLS, UI state, and Runtime in-car button integration. React, Vue, and vanilla pages all use it through the same `getSnapshot()` / `subscribe()` interface.

The player requires the page to run in a Toooony Runtime that provides `AndroidCarBridge`. If the Runtime does not provide this interface, the player throws `MULTIMEDIA_BRIDGE_UNAVAILABLE`. Do not create or call `window.AndroidCarBridge` or `window.__carBridge` directly, and do not listen for internal Runtime media events.

```html
<button id="play-music" type="button">Play</button>
```

```ts
import { MultimediaPlayer } from '@ziztechnology/dial-library';

const tracks = [
  { id: 1, title: 'First Song', artist: 'Toooony', url: 'https://cdn.example.com/first.mp3' },
  { id: 2, title: 'Second Song', artist: 'Toooony', url: 'https://cdn.example.com/second.m3u8' },
];

const audio = new Audio();
audio.preload = 'auto';

const player = new MultimediaPlayer(audio, {
  tracks,
  playbackMode: 'repeat-all',
});

const unsubscribe = player.subscribe((snapshot) => {
  console.log(snapshot.currentTrack?.title, snapshot.phase, snapshot.positionMs, snapshot.error);
});

const playButton = document.querySelector<HTMLButtonElement>('#play-music');
if (!playButton) throw new Error('Could not find #play-music');

// Request initial playback from a real user gesture. The SDK writes asynchronous failures to snapshot.error.
playButton.addEventListener('click', () => player.play());

let resumeAfterPageShow = false;

window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    resumeAfterPageShow = player.getSnapshot().isPlayRequested;
    player.pause();
    return;
  }

  unsubscribe();
  player.destroy();
});

window.addEventListener('pageshow', (event) => {
  if (!event.persisted || !resumeAfterPageShow) return;
  resumeAfterPageShow = false;
  player.play();
});
```

## Configure Tracks and Playback Modes

Each track must provide `id`, `title`, and `url`. `artist` and `artworkUrl` are optional. Do not modify track objects after creating the player. To replace the entire list, destroy the player and create a new one.

Use `initialIndex` to select the initial track. A non-empty list starts at index `0` by default. An index outside the playlist range throws `RangeError`. For an empty list, omit `initialIndex` or pass `0`; the player's `currentIndex` is then `-1`.

The player provides the following playlist modes:

| `playbackMode` | Behavior                                                                    |
| -------------- | --------------------------------------------------------------------------- |
| `repeat-all`   | Default; plays in order and returns to the first track after the last track |
| `shuffle`      | Randomly selects a track other than the current one and retains history for `previous()` |
| `once`         | Does not loop; stops at list boundaries and disables the corresponding previous or next control |

Public control methods include `play()`, `pause()`, `toggle()`, `next()`, `previous()`, `playAt(index)`, and `seekToMs(positionMs)`. An empty list remains `idle`; valid playback controls other than `playAt(index)` do not throw. An invalid index passed to `playAt()` or invalid millisecond value passed to `seekToMs()` throws `RangeError`.

## Synchronize UI State

`getSnapshot()` returns the complete current state. `subscribe(listener)` notifies only subsequent changes and returns an idempotent unsubscribe function.

| Field                 | Meaning                                                                       |
| --------------------- | ----------------------------------------------------------------------------- |
| `lifecycle`           | `active` or `destroyed`                                                       |
| `phase`               | `idle`, `loading`, `paused`, `playing`, `buffering`, or `error`              |
| `currentTrack`        | Current track; `null` for an empty list                                       |
| `currentIndex`        | Current index; `-1` for an empty list                                         |
| `isPlayRequested`     | Whether the user or Runtime requested continued playback                     |
| `isActuallyPlaying`   | Whether the audio element has emitted `playing`                              |
| `positionMs`          | Current playback position                                                     |
| `durationMs`          | Total audio duration; `null` when unavailable or for a live stream           |
| `canNext`             | Whether the player can currently switch to the next track                    |
| `canPrevious`         | Whether the player can currently switch to the previous track                |
| `error`               | `{ category, message, trackId }`; `null` under normal conditions             |

The SDK does not automatically pause, resume, or destroy music on `document.hidden` or `pagehide`. The application must call `pause()` or `destroy()` based on page state. When the player is destroyed, the SDK pauses audio, clears the source that it set, and releases HLS, listeners, timers, and the Runtime control connection.

## Play HLS

When playing a `.m3u8` audio source, ensure the manifest, media segments, and encryption keys permit cross-origin access from the current page. Adjust HLS configuration through `hlsConfig`. In specialized packaging environments, provide a custom module loader through `loadHls`.

Audio sources requiring authentication must use a trusted URL that the browser can access directly or integrate an application-specific loading strategy through `hlsConfig` / `loadHls`. Use `errorAdvanceDelayMs` to set the delay before switching tracks after a playback failure, and display or record errors from the subscribed `snapshot.error`.

## Integrate an Existing Custom Player

An existing custom player may continue to use `MultimediaController` by providing only the state and control callbacks required by the Runtime:

```ts
import { MultimediaController } from '@ziztechnology/dial-library';

const controller = new MultimediaController({
  getState: () => ({
    isPlaying: !audio.paused && !audio.ended,
    title: currentTrack.title,
    artist: currentTrack.artist ?? '',
    canNext: true,
    canPrevious: true,
  }),
  controls: {
    play: () => audio.play(),
    pause: () => audio.pause(),
    next: () => playNextTrack(),
    previous: () => playPreviousTrack(),
    seekToMs: (positionMs) => {
      audio.currentTime = positionMs / 1_000;
    },
  },
});

const reportState = () => {
  controller.reportState();
};

audio.addEventListener('playing', reportState);
audio.addEventListener('pause', reportState);

let resumeAfterPageShow = false;

window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    resumeAfterPageShow = !audio.paused && !audio.ended;
    audio.pause();
    controller.reportState();
    return;
  }

  audio.removeEventListener('playing', reportState);
  audio.removeEventListener('pause', reportState);
  audio.pause();
  controller.reportState();
  controller.destroy();
});

window.addEventListener('pageshow', (event) => {
  if (!event.persisted || !resumeAfterPageShow) return;
  resumeAfterPageShow = false;
  void audio.play().then(reportState, reportState);
});
```

`getState()` must return `isPlaying`, `title`, `artist`, `canNext`, and `canPrevious` immediately. It cannot return a Promise. Control callbacks may return Promises. Call `reportState()` whenever the player's actual playback state changes.

Only one `MultimediaPlayer` or `MultimediaController` may exist on a page at a time. Distinguish initialization errors through `MultimediaBridgeError.code`:

| Code                                      | Meaning                                                        |
| ----------------------------------------- | -------------------------------------------------------------- |
| `MULTIMEDIA_BRIDGE_UNAVAILABLE`           | The Runtime does not provide an available media Bridge         |
| `MULTIMEDIA_BRIDGE_CONFLICT`              | A media command entry point or another controller already exists on the page |
| `MULTIMEDIA_BRIDGE_INITIALIZATION_FAILED` | The Ready notification or initial state report failed          |

Use `reportIntervalMs` to adjust the playback state reporting interval. Call `destroy()` at the end to disconnect the in-car buttons and release resources.

## Debug Logs

When troubleshooting, filter console logs for entries beginning with `[DIAL_SDK_DEBUG]`.

Application code has no interface for disabling logs at runtime. In production integrations, remember that logs may contain device state and media URLs.
