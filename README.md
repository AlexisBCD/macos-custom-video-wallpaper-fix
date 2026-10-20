# Fix: custom video wallpaper goes **black after unlock** on macOS (Sonoma / Sequoia / Tahoe)

`./wallpaper` downloads parts of a YouTube video and turns them into a macOS custom video wallpaper that keeps playing after unlock.

## Usage

`wallpaper` works by replacing one of Apple's own aerial wallpapers with your video. macOS
only has the file once you've selected that aerial: open **System Settings → Wallpaper**,
pick an aerial wallpaper and wait for it to download. It is then stored as
`~/Library/Application Support/com.apple.wallpaper/aerials/videos/<UUID>.mov`, and the new
video is made to match its resolution and duration.

Find the `<UUID>` with:

```bash
./wallpaper list
# 4C108785-A7BA-422E-9C79-B0129F1D5550  3840x2160    300.0s  2026-10-19 21:05
```

The most recently modified entry is usually the aerial you just picked. If you have
several, tell them apart by size and duration. If the list is empty, select an aerial in
System Settings first.

```
./wallpaper make YOUTUBEID UUID [--loop] START END [START END ...]
./wallpaper list
./wallpaper verify file.mov
```

```bash
# one stretch of the video, slowed/sped to fit the wallpaper's duration
./wallpaper make rVyQHMVhkrE 4C108785-A7BA-422E-9C79-B0129F1D5550 00:05 04:29

# several stretches joined, played at normal speed and repeated to fill the duration
./wallpaper make rVyQHMVhkrE 4C108785-A7BA-422E-9C79-B0129F1D5550 --loop \
    00:05 00:20  01:10 01:35  04:00 04:29
```

- `YOUTUBEID` — a video id or a full URL. Downloads are cached in `~/Documents/desktops`
  (`--cache-dir` to change).
- `UUID` — the wallpaper to replace; a path ending in `.mov` also works.
- `START END` — `MM:SS` or `HH:MM:SS` pairs; the pieces are joined in order.
- `--loop` — keep original speed and repeat the cut until the target duration is filled
  (the default stretches the cut to the target duration instead).
- `--bitrate MBPS` — default `12` (Apple's aerials are ~10–12 Mbps 4K).

The result is verified for the `tscl`/`tsas` sample groups *before* it replaces the
existing wallpaper, so a failed run never clobbers a working one.

---

## The problem

Turn any video into a **custom aerial / live wallpaper** that keeps playing after you
unlock your Mac — instead of turning into a **black screen**.

The real root cause and a working, self-contained encoder (Swift + VideoToolbox) are in
this repo. As far as I can tell, **this is the first place the actual cause is documented.**

---

## Does this match your problem?

If you've hit any of these, you're in the right place:

- You added your own `.mov` as an animated/aerial wallpaper and the desktop **turns
  black after you unlock** (or after wake from sleep / screensaver).
- The wallpaper **plays once, then freezes or goes black** and never animates again.
- Custom **dynamic / live / aerial wallpaper stops animating** on macOS 14 Sonoma,
  15 Sequoia, or 26 Tahoe.
- You used a helper app such as **VideoPaper**, or dropped a file into
  `~/Library/Application Support/com.apple.wallpaper/aerials/videos/`, and it doesn't
  stick.
- Console shows:
  ```
  [video-player] Error handling sample: Error Domain=WallpaperExtensionKit.VideoSampleReadingErrors Code=4.
  [video-player-layer] VideoPlayerLayer failed to acquire next sample due to error: ...Code=4
  ```

There is **no crash report** — the video decodes fine, it just never composites, so the
desktop is black.

## TL;DR — the fix

Apple's built-in aerials are **HEVC videos with temporal scalability** (multiple HEVC
**temporal sub-layers**, described by `tscl` / `tsas` sample groups). The wallpaper
engine's unlock animation ("ramp down" — the smooth slow‑motion‑to‑freeze effect) reads
each frame's temporal-layer info. If it's missing, it throws
`VideoSampleReadingErrors.noTemporalInfo` (**Code=4**) and shows black.

**No public encoder produces this** — not `ffmpeg` (`libx265` or `hevc_videotoolbox`),
not Apple's own `avconvert`. You have to drive **VideoToolbox** directly with
`kVTCompressionPropertyKey_BaseLayerFrameRate` on the hardware encoder. That's exactly
what [`encode_temporal.swift`](encode_temporal.swift) does.

```bash
# one command: download -> cut -> encode with temporal layers -> verify -> install
./wallpaper make YOUTUBEID WALLPAPER-UUID 00:05 04:29
```

Then lock/unlock — it keeps playing. ✅

## Requirements

- macOS 26 (Tahoe) — developed/tested on 26.4.1 (build 25E253), Apple Silicon.
  The same mechanism exists on Sonoma/Sequoia; adjust as needed.
- [`uv`](https://docs.astral.sh/uv/) — runs `./wallpaper` and installs its only Python
  dependency (`yt-dlp`) automatically; there is nothing to `pip install`.
- `ffmpeg` / `ffprobe` (`brew install ffmpeg`).
- Xcode command-line tools (`swiftc`). The Swift encoder is compiled automatically the
  first time it is needed (and again whenever `encode_temporal.swift` changes).

## Using the encoder on its own

```bash
swiftc -O encode_temporal.swift -o encode_temporal
./encode_temporal <input.mov> <output.mov> [loopCount] [bitrateMbps]
```

- `loopCount` — repeat the source content back-to-back with continuous timestamps
  (default `1`).
- `bitrateMbps` — average bitrate (default `10`).

The encoder:
1. decodes the source with `AVAssetReader` (to 10-bit pixel buffers),
2. re-encodes with `VTCompressionSession` (HEVC **Main 10 / 10-bit**, hardware,
   **2 temporal sub-layers** via `BaseLayerFrameRate = fps/2`), writing a BT.709 `colr` atom,
3. writes via `AVAssetWriter`, which emits the `tscl` / `tsas` sample groups.

> **Why 10-bit (Main 10)?** Apple's aerials are 10-bit. If you encode 8-bit, playback
> and the unlock ramp still work, but the **preview thumbnail** in System Settings →
> Wallpaper renders as garbled black‑and‑white noise (the poster generator expects
> 10-bit). Encoding Main 10 fixes the preview. The included encoder outputs 10-bit by
> default.

## The looping caveat (make it long)

There's a **second, separate** issue: the wallpaper's looping is not robust for **short**
clips. An ~18 s clip plays, loops once, then freezes (`rate 0.00`, permanent black); the
log shows `Error copying sample buffer. Buffer was nil.` at the loop boundary. Apple's
built-in aerials are long (~5–8 min) so they essentially never loop during a session.

That's why the encoder has a `loopCount` (and `wallpaper make` matches the duration of the
wallpaper it replaces): make a **long** file (≈5 min is plenty and matches Apple) so
looping doesn't happen in normal use:

```bash
./encode_temporal source.mov out.mov 17 12   # 18s x17 ≈ 5 min, ~460 MB
```

## Install as a wallpaper

`./wallpaper make` installs straight into the aerials folder. For a file produced by hand:

**Option A — VideoPaper (or similar helper app):** just import `out.mov`.

**Option B — manual:** replace the `.mov` the aerial system already references under
`~/Library/Application Support/com.apple.wallpaper/aerials/videos/<UUID>.mov`.

> **Don't restart `WallpaperAgent`** (e.g. `pkill -x WallpaperAgent`) after replacing the
> file: macOS then re-downloads Apple's original video and overwrites yours.

It is unclear how to make macOS pick up the replaced video straight away. Things to try
(untested): switch to a different wallpaper in System Settings and back, or just wait.

## Verify it worked

```bash
./wallpaper verify out.mov
# sgpd 'tscl' + sgpd 'tsas' + csgm 'tscl' + csgm 'tsas'  == same as a genuine Apple aerial
```

After installing, a clean unlock logs the ramp completing with **zero** `Code=4`:

```
[video-player] Switching player state <private> -> .beginRampingDown
[video-player] Switching player state .beginRampingDown -> <private>   # ramp completes
AVSampleBufferVideoRenderer setReadyForDisplay ... YES                  # stays visible
```

---

## Deep dive: how the root cause was found

The playback runs in
`/System/Library/ExtensionKit/Extensions/WallpaperAerialsExtension.appex`, linking the
private `WallpaperExtensionKit.framework` (only present in the dyld shared cache). Steps:

1. The fatal log is `WallpaperExtensionKit.VideoSampleReadingErrors Code=4` during the
   unlock transition (`Presentation Mode 'locked' -> 'default'` → `.beginRampingDown`).
2. Extract the framework from the shared cache with `/usr/lib/dsc_extractor.bundle`,
   parse its Swift `__swift5_fieldmd` metadata → the enum case order:

   | Code | Case |
   |---|---|
   | 0 | `noAssetReader` |
   | 1 | `noSampleBuffer` |
   | 2 | `noPixelBuffer` |
   | 3 | `noTimeAdjustedSampleBuffer` |
   | **4** | **`noTemporalInfo`** |

3. The framework imports `kCMSampleAttachmentKey_HEVCTemporalLevelInfo`,
   `...HEVCTemporalSubLayerAccess`, `...HEVCStepwiseTemporalSubLayerAccess`.
4. Comparing MP4 sample groups confirms it — Apple aerials have temporal-scalability
   sample groups; ordinary encodes have none:
   ```
   Apple aerial:   sgpd 'tscl'  sgpd 'tsas'  csgm 'tscl'  csgm 'tsas'
   Normal encode:  (none)
   ```

The unlock animation slows the video by dropping the higher HEVC temporal layer; with no
temporal layers there's nothing to drop and no per-frame temporal info → `noTemporalInfo`
→ black.

## What did NOT fix it (so you don't waste time)

All of these were tried and made **no** difference:

| Attempt | Result |
|---|---|
| Container / filename / UUID / manifest catalog entry | ❌ (Apple's *bytes* work under any name) |
| Color atom `colr` / BT.709 | ❌ |
| Track aperture `tapt` | ❌ |
| 8-bit vs 10-bit (Main vs Main10) | ❌ for the black-screen bug — but 10-bit *is* needed to fix the preview thumbnail, see below |
| B-frames / `ctts` | ❌ |
| Frame rate 24 → 240 fps, timescale 240000 | ❌ |
| `sdtp` / `cslg` sample tables | ❌ |
| `MADMLEnhancementRequest ... "Could not obtain ML enhancement result"` | ❌ red herring — genuine Apple aerials log it too |
| Reduce Motion accessibility setting | ❌ |
| Re-encoding with Apple's own `avconvert` | ❌ |
| **HEVC temporal sub-layers (`tscl`/`tsas`) via VideoToolbox** | ✅ **this is the fix** |

## Files

- [`wallpaper`](wallpaper) — the single entry point (`uv run --script`): download, cut, encode, verify, install.
- [`encode_temporal.swift`](encode_temporal.swift) — the VideoToolbox temporal-scalability encoder, built on demand by `wallpaper`.

## Credits

Thank you to Alexis ([AlexisBCD](https://github.com/AlexisBCD)), who tracked down the real
cause of the black screen, built the VideoToolbox encoder (`encode_temporal.swift`) and
shared it all openly. The root-cause analysis and the encoder in this repo are their work;
the `wallpaper` command just wraps them together with the download and cutting steps.

## License

MIT — see [LICENSE](LICENSE).

---

<sub>Keywords: macOS custom video wallpaper black screen, aerial wallpaper black after
unlock, custom live wallpaper stops animating, dynamic wallpaper black desktop, macOS
Sonoma / Sequoia / Tahoe wallpaper not working, VideoPaper black screen,
WallpaperAerialsExtension, VideoSampleReadingErrors Code 4, noTemporalInfo, HEVC temporal
sub-layers / temporal scalability, VideoToolbox BaseLayerFrameRate, tscl tsas sample
groups, com.apple.wallpaper aerials videos.</sub>
