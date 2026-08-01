# Sticker Carousel

## Create a Sticker Carousel

Use `StickerCarousel` to display remote static image, TGS, or WebM Stickers in random order.

Provide the carousel with a dedicated container that has explicit dimensions. For example:

```html
<div id="sticker-stage"></div>

<style>
  #sticker-stage {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }
</style>
```

If you use a percentage height, the parent must also have a computable height.

```ts
import {
  StickerCarousel,
  parseStickerCarouselConfig,
} from "@ziztechnology/dial-library";
import rawStickerConfig from "./stickers.json";

const container = document.querySelector<HTMLElement>("#sticker-stage");
if (!container) throw new Error("Could not find #sticker-stage");

const config = parseStickerCarouselConfig(rawStickerConfig);
const carousel = new StickerCarousel(container, config, {
  initialCoverPriorityMs: 2_000,
  onError(error, sticker) {
    console.warn(`Failed to load Sticker ${sticker.name}.`, error);
  },
});

carousel.start();

const destroy = () => {
  carousel.destroy();
};

window.addEventListener("pagehide", (event) => {
  if (!event.persisted) destroy();
});
```

The configuration format is fixed at `schemaVersion: 2`:

```json
{
  "schemaVersion": 2,
  "intervalMs": 5000,
  "playbackMode": "shuffle",
  "fit": "contain",
  "stickers": [
    {
      "name": "Sticker 001",
      "position": 0,
      "kind": "tgs",
      "url": "https://cdn.example.com/sticker-001.tgs",
      "sizeBytes": 32768,
      "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "coverUrl": "https://cdn.example.com/sticker-001.webp",
      "coverSizeBytes": 8192,
      "coverSha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
    }
  ]
}
```

`kind` supports `image`, `tgs`, and `webm`; `image` supports WebP and PNG. `sizeBytes` is measured in bytes and must be a safe integer from `1` to `16,777,216`. `sha256` must be a 64-character hexadecimal string.

`playbackMode` currently supports only `shuffle`, and `fit` currently supports only `contain`. They are not freely selectable display options. Passing other values throws `StickerConfigError`.

`coverUrl`, `coverSizeBytes`, and `coverSha256` must either all be provided or all be omitted. In a standard `StickerCarousel` configuration, every primary media URL and cover URL must be unique, so multiple Stickers cannot share the same cover. Media must use long-lived, public HTTPS URLs.

`parseStickerCarouselConfig()` and `parseStickerSetCarouselConfig()` both throw `StickerConfigError` for invalid configuration. If the configuration comes from an API or user input, display the error with `try/catch` instead of continuing to create the carousel.

Set `initialCoverPriorityMs` to prioritize displaying covers during a cold start. The cover loads exclusively for the specified duration; after that duration, the primary media begins loading in parallel. The default value of `0` loads the cover and primary media in parallel.

Use `onError` to record media loading failures. Use `onMediaCommitted(sticker, mediaKind)` to update UI related to the current media. `mediaKind` may be `image`, `cover`, `tgs`, or `webm`. Call `pause()` and `resume()` to control temporary suspension, and call `destroy()` to release resources permanently. Set `managePageLifecycle: false` when the page already has centralized lifecycle management.

Set `loadTimeoutMs`, `videoLoadTimeoutMs`, `maxTgsCompressedBytes`, `maxTgsJsonBytes`, `maxTgsLayers`, and `maxPrefetchBytes` as needed to control loading timeouts and size limits. You may also pass custom `fetch` and random-number generator functions.

Sticker TGS media must also meet the preceding [TGS media requirements for the driving expression player](driving-expressions.md#tgs-media-requirements).

## Display Stickers by Set

To group Stickers into sets first and then display Stickers within each set, use `StickerSetCarousel` with `parseStickerSetCarouselConfig()`. A set configuration does not require `playbackMode` and has the following format:

```json
{
  "schemaVersion": 2,
  "intervalMs": 5000,
  "fit": "contain",
  "sets": [
    {
      "name": "funny-cats",
      "title": "Funny Cats",
      "stickerType": "regular",
      "stickers": [
        {
          "name": "Cat 001",
          "position": 0,
          "kind": "webm",
          "url": "https://cdn.example.com/cat-001.webm",
          "sizeBytes": 65536,
          "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
          "coverUrl": "https://cdn.example.com/cats-cover.webp",
          "coverSizeBytes": 8192,
          "coverSha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
        }
      ]
    }
  ]
}
```

```ts
import {
  StickerSetCarousel,
  parseStickerSetCarouselConfig,
} from "@ziztechnology/dial-library";
import rawSetConfig from "./sticker-sets.json";

const container = document.querySelector<HTMLElement>("#sticker-stage");
if (!container) throw new Error("Could not find #sticker-stage");

const config = parseStickerSetCarouselConfig(rawSetConfig);
const carousel = new StickerSetCarousel(container, config, {
  initialCoverPriorityMs: 2_000,
  onMediaCommitted(sticker, mediaKind) {
    console.log(`Committed ${mediaKind}: ${sticker.name}`);
  },
  onSetChange(set) {
    console.log(`Started playing set: ${set.title}`);
  },
});

carousel.start();

window.addEventListener("pagehide", (event) => {
  if (!event.persisted) carousel.destroy();
});
```

A set's `name` and `title` must be non-empty strings. Within one configuration, `name` values must be unique when compared case-insensitively. `stickerType` may be omitted; when omitted, it becomes an empty string. Each set's `stickers` must be a non-empty array. Primary media URLs must be unique within a single set, but covers may be shared, and different sets may reference the same primary media. The top-level `sets` array may be empty, in which case the carousel displays no content.

`StickerSetCarousel` applies `initialCoverPriorityMs` only while the entire set carousel has no visible media. When visible media already exists, switching sets loads the cover and primary media in parallel. It calls `onMediaCommitted` after completing the internal visible-carousel transition.

Use `updateConfig()` to update the configuration during playback. If a set is playing, a non-empty new configuration takes effect after the current set finishes. `sets: []` immediately stops current playback and clears the container. Call `destroy()` when the carousel is no longer needed.
