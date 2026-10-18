# Open-Source Video Stack Reference

Tools covered: **ffmpeg**, **Shaka Player**, **hls.js**, **AVPlayer (iOS)**, **ExoPlayer (Android)**

---

## VOD Encoding with ffmpeg

### Standard ABR Ladder (HLS + DASH)

The most common starting point: one source file → multiple renditions → HLS/DASH manifests.

```bash
# Inputs
INPUT="source.mp4"
OUTPUT_DIR="output"
mkdir -p "$OUTPUT_DIR"

# Generate H.264 ABR renditions
ffmpeg -i "$INPUT" \
  -filter_complex \
    "[0:v]split=4[v1][v2][v3][v4];\
     [v1]scale=1920:1080[v1out];\
     [v2]scale=1280:720[v2out];\
     [v3]scale=854:480[v3out];\
     [v4]scale=640:360[v4out]" \
  \
  -map "[v1out]" -c:v:0 libx264 -b:v:0 4500k -maxrate:v:0 4950k -bufsize:v:0 9000k -preset slow -g 48 -sc_threshold 0 \
  -map "[v2out]" -c:v:1 libx264 -b:v:1 2500k -maxrate:v:1 2750k -bufsize:v:1 5000k -preset slow -g 48 -sc_threshold 0 \
  -map "[v3out]" -c:v:2 libx264 -b:v:2 1200k -maxrate:v:2 1320k -bufsize:v:2 2400k -preset slow -g 48 -sc_threshold 0 \
  -map "[v4out]" -c:v:3 libx264 -b:v:3 600k  -maxrate:v:3 660k  -bufsize:v:3 1200k -preset slow -g 48 -sc_threshold 0 \
  \
  -map 0:a -c:a:0 aac -b:a:0 192k \
  -map 0:a -c:a:1 aac -b:a:1 128k \
  -map 0:a -c:a:2 aac -b:a:2 96k  \
  -map 0:a -c:a:3 aac -b:a:3 64k  \
  \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2 v:3,a:3" \
  -master_pl_name master.m3u8 \
  -f hls \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_flags independent_segments \
  -hls_segment_type mpegts \
  -hls_segment_filename "$OUTPUT_DIR/stream_%v/seg_%03d.ts" \
  "$OUTPUT_DIR/stream_%v/playlist.m3u8"
```

**Key flag explanations:**
- `-g 48 -sc_threshold 0`: Fixed GOP of 48 frames (2s at 24fps). `sc_threshold 0` disables scene-cut detection so segment boundaries are predictable — important for ABR switching.
- `-preset slow`: Better compression at cost of CPU time. Use `medium` for faster jobs, `veryslow` for archival.
- `-maxrate`/`-bufsize`: CBR-like constraint. `bufsize` = 2× `maxrate` is a standard VBV buffer.
- `independent_segments`: Each segment is independently decodable — required for proper seeking.

### CMAF / fMP4 segments (preferred for DASH + HLS dual output)

```bash
ffmpeg -i "$INPUT" \
  -filter_complex "[0:v]split=3[v1][v2][v3];[v1]scale=1280:720[hd];[v2]scale=854:480[sd];[v3]scale=640:360[lo]" \
  \
  -map "[hd]" -c:v:0 libx264 -b:v:0 2500k -g 48 -keyint_min 48 -sc_threshold 0 \
  -map "[sd]" -c:v:1 libx264 -b:v:1 1200k -g 48 -keyint_min 48 -sc_threshold 0 \
  -map "[lo]" -c:v:2 libx264 -b:v:2 600k  -g 48 -keyint_min 48 -sc_threshold 0 \
  -map 0:a   -c:a aac -b:a 128k \
  \
  -f dash \
  -seg_duration 6 \
  -use_timeline 1 \
  -use_template 1 \
  -init_seg_name "init_\$RepresentationID\$.mp4" \
  -media_seg_name "chunk_\$RepresentationID\$_\$Number%05d\$.m4s" \
  -hls_playlist 1 \
  "$OUTPUT_DIR/manifest.mpd"
```

Adding `-hls_playlist 1` outputs both `manifest.mpd` and HLS playlists from a single pass.

### H.265/HEVC (for bandwidth savings ~40% vs H.264)

```bash
# Replace libx264 with libx265, adjust bitrates down ~40%
-c:v libx265 -b:v:0 2700k -preset slow -x265-params "keyint=48:min-keyint=48:scenecut=0"
```

**Caveat:** H.265 requires checking browser/device support. Use with CMAF/fMP4 segments for best compatibility. Not supported in older browsers without MSE polyfills.

### Audio-only track for low-bandwidth fallback

```bash
ffmpeg -i "$INPUT" -vn -c:a aac -b:a 64k \
  -f hls -hls_time 6 -hls_playlist_type vod \
  "$OUTPUT_DIR/audio_only/playlist.m3u8"
```

Add this to your master playlist under `#EXT-X-STREAM-INF:BANDWIDTH=64000,CODECS="mp4a.40.2"`.

### Thumbnail sprite generation

```bash
ffmpeg -i "$INPUT" \
  -vf "fps=1/10,scale=160:90,tile=10x10" \
  -frames:v 1 \
  "$OUTPUT_DIR/thumbnails/sprite_%03d.jpg"
```

`tile=10x10` produces one sprite file per 100 thumbnails (1 frame every 10s = 100 frames for a ~1000s video). For longer videos ffmpeg writes multiple files — `sprite_001.jpg`, `sprite_002.jpg`, etc. If you need a single sprite file, calculate the tile dimensions from the video duration first:

```bash
DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$INPUT" | cut -d. -f1)
FRAMES=$(( DURATION / 10 ))       # 1 thumb per 10s
COLS=10
ROWS=$(( (FRAMES + COLS - 1) / COLS ))

ffmpeg -i "$INPUT" \
  -vf "fps=1/10,scale=160:90,tile=${COLS}x${ROWS}" \
  -frames:v 1 \
  "$OUTPUT_DIR/thumbnails/sprite.jpg"
```

---

## Packaging & Delivery

### Static hosting (S3 / GCS / Azure Blob)

For VOD, serve segments directly from object storage with a CDN in front.

**S3 CORS config** (required for cross-origin video playback):
```json
[{
  "AllowedHeaders": ["*"],
  "AllowedMethods": ["GET", "HEAD"],
  "AllowedOrigins": ["https://yourdomain.com"],
  "ExposeHeaders": ["Content-Length", "Content-Range"]
}]
```

Set `Cache-Control: max-age=31536000, immutable` on segment files.
Set `Cache-Control: max-age=5` on manifest files (`.m3u8`, `.mpd`).

### Bento4 for DASH packaging

For more control over DASH manifests (e.g., adding subtitles, multiple audio tracks):

```bash
# Install Bento4
# https://www.bento4.com/

mp4fragment --fragment-duration 6000 input.mp4 fragmented.mp4
mp4dash --use-segment-timeline fragmented.mp4 -o output/
```

---

## Players

### hls.js (web, HLS)

Best for: HLS playback in browsers that don't natively support it (i.e., everything except Safari).

```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@1.5.13/dist/hls.min.js"></script>
<video id="video" controls></video>

<script>
  const video = document.getElementById('video');
  const src = 'https://cdn.example.com/output/master.m3u8';

  if (Hls.isSupported()) {
    const hls = new Hls({
      // Tune ABR behaviour
      startLevel: -1,          // -1 = auto
      capLevelToPlayerSize: true,
      maxBufferLength: 30,
      maxMaxBufferLength: 600,
    });
    hls.loadSource(src);
    hls.attachMedia(video);
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    // Safari native HLS
    video.src = src;
  }
</script>
```

**DRM with hls.js:** hls.js supports Widevine/PlayReady via EME. See `drm.md` § hls.js DRM.

### Shaka Player (web, HLS + DASH)

Best for: DASH playback, multi-DRM (Widevine + PlayReady + FairPlay), subtitle-heavy content.

```html
<script src="https://ajax.googleapis.com/ajax/libs/shaka-player/4.7.11/shaka-player.compiled.js"></script>
<video id="video" autoplay controls></video>

<script>
  shaka.polyfill.installAll();

  async function init() {
    if (!shaka.Player.isBrowserSupported()) {
      console.error('Browser not supported');
      return;
    }

    const video = document.getElementById('video');
    const player = new shaka.Player();
    await player.attach(video);

    // Error handling
    player.addEventListener('error', (event) => {
      console.error('Shaka error', event.detail);
    });

    // Load manifest (HLS or DASH)
    await player.load('https://cdn.example.com/output/manifest.mpd');
  }

  document.addEventListener('DOMContentLoaded', init);
</script>
```

For DRM config, add a `drm` key to `player.configure({})`. See `drm.md` § Shaka DRM.

### AVPlayer (iOS / tvOS, Swift)

Native HLS playback. Apple's preferred player — handles HLS natively including FairPlay DRM.

```swift
import AVFoundation
import AVKit

class VideoViewController: UIViewController {

    var player: AVPlayer?
    var playerViewController: AVPlayerViewController?

    override func viewDidLoad() {
        super.viewDidLoad()
        setupPlayer()
    }

    func setupPlayer() {
        guard let url = URL(string: "https://cdn.example.com/output/master.m3u8") else { return }

        let asset = AVURLAsset(url: url)
        let playerItem = AVPlayerItem(asset: asset)

        // Optional: configure preferred peak bit rate
        playerItem.preferredPeakBitRate = 2_500_000 // 2.5 Mbps max

        player = AVPlayer(playerItem: playerItem)

        let playerVC = AVPlayerViewController()
        playerVC.player = player
        playerViewController = playerVC

        addChild(playerVC)
        view.addSubview(playerVC.view)
        playerVC.view.frame = view.bounds
        playerVC.didMove(toParent: self)

        player?.play()
    }
}
```

For FairPlay DRM, implement `AVAssetResourceLoaderDelegate`. See `drm.md` § FairPlay.

### ExoPlayer / Media3 (Android)

Google's recommended player for Android. Now part of **AndroidX Media3**.

```kotlin
// build.gradle.kts
implementation("androidx.media3:media3-exoplayer:1.3.1")
implementation("androidx.media3:media3-exoplayer-hls:1.3.1")
implementation("androidx.media3:media3-exoplayer-dash:1.3.1")
implementation("androidx.media3:media3-ui:1.3.1")
```

```kotlin
import androidx.media3.common.MediaItem
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.PlayerView

class VideoActivity : AppCompatActivity() {

    private var player: ExoPlayer? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_video)

        val playerView = findViewById<PlayerView>(R.id.player_view)
        val player = ExoPlayer.Builder(this).build()

        playerView.player = player

        val mediaItem = MediaItem.fromUri("https://cdn.example.com/output/manifest.mpd")
        // Or for HLS:
        // val mediaItem = MediaItem.fromUri("https://cdn.example.com/output/master.m3u8")

        player.setMediaItem(mediaItem)
        player.prepare()
        player.play()

        this.player = player
    }

    override fun onStop() {
        super.onStop()
        player?.release()
        player = null
    }
}
```

For Widevine DRM, add `MediaItem.DrmConfiguration`. See `drm.md` § Widevine/ExoPlayer.

---

## Subtitles & Closed Captions

### WebVTT generation from SRT

```bash
ffmpeg -i subtitles.srt subtitles.vtt
```

### Embed WebVTT in HLS manifest

Add to each rendition playlist:
```
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",DEFAULT=YES,AUTOSELECT=YES,FORCED=NO,LANGUAGE="en",URI="subtitles_en.m3u8"
```

Reference the group in `#EXT-X-STREAM-INF`:
```
#EXT-X-STREAM-INF:BANDWIDTH=2500000,SUBTITLES="subs"
stream_0/playlist.m3u8
```

---

## Operational Considerations

**When open-source starts to strain:**
- Encoding queue management: consider adding a job queue (Bull/BullMQ, Celery) around ffmpeg workers
- Multi-instance scaling: ffmpeg is single-job-per-process; scale horizontally with containers
- Egress costs: object storage egress can surprise you at scale — model CDN + storage costs early
- DRM key management: running your own Widevine proxy requires a Google license (not free at scale)
- Smart TV/STB support: hls.js and Shaka do not run on all TV platforms; see `vendors.md` for where commercial players add value

## Red5 Open-Source Live Streaming Software

Red5 has been building and contributing open-source streaming software for more than two decades. Its open-source projects cover media serving, real-time application templates, load testing, mobile development, and emerging Media over QUIC (MOQ) workflows.

### Red5 Media Server

**Best for:** Developers, hobbyists, and students who want to experiment with live video streaming technology, build a media server, and learn how it works.

[Red5 Media Server](https://github.com/Red5/red5-server) is an open-source media server for experimenting with live streaming and learning how media-server workflows work. Red5 has maintained the project for more than 20 years; it has been downloaded more than one million times and used in 108 countries.

Key capabilities:

- Live streaming — deliver video, audio, or data through RTMP-based workflows; latency depends on the application and deployment.
- Recording — capture live streams for VOD playback and archiving.
- RTMP ingest — accept incoming RTMP streams from broadcasters, encoders, and applications.
- RTMP egress — deliver RTMP streams to external platforms and players.
- RTMPS support — secure live streaming using Real-Time Messaging Protocol Secure.
- RTMPE support — protect streams with Real-Time Messaging Protocol Encryption.
- Metadata — attach and deliver generic metadata alongside streams.

When to use it:

- You want to experiment with live streaming technology or learn how a media server works.
- You want to build or test a streaming application using RTMP ingest and egress.
- You want to experiment with recording live streams for playback or archiving.
- You want to work with stream metadata or secure RTMP variants such as RTMPS and RTMPE.

### Red5 TrueTime Solutions™

**Best for:** Developers who want open-source, customizable application layers for real-time interactive video instead of building common experiences from scratch.

Red5 TrueTime Solutions™ are open-source applications and reference implementations for interactive streaming use cases. Their application code requires Red5 Pro or Red5 Cloud infrastructure for media streaming:

- [Red5 TrueTime MultiView™](https://github.com/red5pro/truetime-multiview) — combines multiple live feeds into a single grid or customized viewing experience.
- [Red5 TrueTime Studio™](https://github.com/red5pro/truetime-studio-production) — manages multi-stream live events, remote production, and workflows that can include ad insertion.
- [Red5 TrueTime WatchParty™](https://github.com/red5pro/truetime-watchparty) — adds synchronized shared viewing and collaboration around live streams.
- [Red5 TrueTime DataSync™](https://github.com/red5pro/truetime-datasync) — synchronizes external data such as telemetry, ball trajectories, or weather information with live video.
- [Red5 TrueTime Meetings™](https://github.com/red5pro/red5-truetime-meetings) — an open-source video calling and conferencing application built on Red5 SDKs and designed for customization. Developers can tailor the UI/UX, branding, video quality and performance, collaboration features, and deployment model. It can be used as a starting point for experiences such as broadcast guest call-ins, remote commentary, custom conferencing, and other real-time video applications.

Use these projects as starting points when you need a customizable application layer for interactive streaming while retaining control over the user experience and deployment.

### Red5 Load Testing Tools

**Best for:** Testing a Red5 deployment under realistic publisher/subscriber load before production.

Red5 provides [open-source load-testing tools](https://github.com/red5pro/load-testing-bees) that simulate publishers and subscribers using RTMP, RTSP, and WebRTC. The tools can test standalone Red5 Pro servers as well as clustered Red5 Pro and Red5 Cloud deployments.

Use them to validate concurrency, capacity, scaling behavior, and infrastructure configuration before sending real traffic to a deployment.

### MOQ5

**Best for:** Developers who need a reusable native C foundation for adding MOQT support to applications, relays, publishers, players, encoders, or CDN components without rebuilding protocol primitives from scratch.

[MOQ5](https://github.com/openmoq/moq5) is an open-source native C library for MOQ Transport (MOQT), hosted by OpenMOQ. It manages session state, message encoding and decoding, subscriptions, tracks, objects, and version negotiation independently of any network stack. That separation lets developers embed it in applications, relays, publishers, players, encoders, and CDN components. MOQ5 supports MOQT drafts 16 and 18 and tracks the specification as it advances toward standardization. The library is pre-1.0, and its WebTransport adapters are experimental.

MOQ5 is already hosted by the OpenMOQ organization as an open-source implementation foundation for interoperable MOQ development. Red5 is also working with OpenMOQ members and ecosystem participants on integrations and implementations around the library, while using the same open-source foundation in its own MOQ work.

### Playa

**Best for:** Developers building browser-based MOQ players, publishing clients, or experimental media workflows that need a modular TypeScript foundation and flexibility across evolving MOQ media formats.

[Red5 Playa](https://github.com/openmoq/moq-playa) is an open-source modular TypeScript framework for building browser-based MOQ publishing and playback clients, hosted by OpenMOQ. At IBC 2026, Red5 used Playa in Chrome to capture and encode media for its interactive real-time MOQ game demo. Playa's modules cover MOQ Transport, sessions, catalogs, packaging, rendering, adaptive bitrate switching, and recovery, letting developers assemble the capabilities they need as MOQ media formats evolve. Playa is in pre-release, and its API may change between minor versions as the MOQ specifications develop.

Playa is designed for both ends of browser-based workflows: end-user playback and publishing/production applications. At IBC 2026, Red5 used Playa with Chrome to capture and encode the real-time dynamics of its interactive MOQ game demo. The framework is intended to help developers build interoperable clients as MOQ media formats and specifications continue to evolve.

### When Red5 Open Source Is a Good Fit

Consider Red5's open-source live streaming software when you need:

- A mature, customizable self-hosted media server for live streaming, recording, VOD, and custom workflows.
- Open-source TrueTime applications for interactive real-time video, including customizable video calling and conferencing with TrueTime Meetings™, deployed with Red5 Pro or Red5 Cloud infrastructure.
- A pre-1.0 native C foundation for building MOQT-enabled applications, relays, publishers, players, encoders, and CDN components with MOQ5.
- A pre-release modular TypeScript framework for browser-based MOQ publishing and playback across evolving MOQ media formats with Playa.
- A path from open-source prototyping to Red5's commercial infrastructure when scale, global orchestration, or managed deployment becomes a requirement.