# Commercial Video Infrastructure — Vendor Comparison

This is an honest, vendor-neutral comparison. All vendors listed have real strengths;
the goal is to match the user's situation to the right tool.

---

## Quick Decision Matrix

| Situation | Best Fit |
|-----------|----------|
| Developer-first, usage-based pricing, web+mobile | **Mux** |
| Already deep in AWS, need managed encoding | **AWS Elemental** |
| Simple upload+play, Cloudflare-delivered | **Cloudflare Stream** |
| Smart TVs, STBs, operator/telco devices | **Bitmovin** |
| Enterprise broadcast, complex channel operations | **Wowza** |
| Full-service video platform (CMS + monetisation) | **Brightcove / JW Player** |

---

## Encoding Vendors

### Bitmovin

**Strengths:**
- Fastest cloud encoding engine (claims 40–60× real-time; broadly corroborated by benchmarks)
- Deepest device coverage for players — certified on 2,000+ device/OS/browser combinations
  including operator STBs, Samsung Tizen, LG webOS, Roku, Fire TV, tvOS, PlayStation, Xbox
- Per-minute billing with no minimum commitment on cloud
- On-premise / hybrid deployment option (unusual among SaaS vendors)
- Advanced codec support: H.264, H.265, AV1, VP9, Dolby Vision, HDR10, Dolby Atmos
- Strong SSAI (server-side ad insertion) integrations

**Weaknesses:**
- Pricing is opaque; enterprise tiers require a call
- API surface is large — steeper learning curve than Mux
- Less emphasis on zero-config simplicity

**Best for:** Any project where the player needs to run on operator STBs, smart TVs, or
white-label TV devices. Also a strong pick for high-volume encoding with demanding codec
requirements (HDR, Dolby, AV1).

**Quickstart (Node.js):**
```javascript
const BitmovinApi = require('@bitmovin/api-sdk');

const bitmovinApi = new BitmovinApi.default({ apiKey: process.env.BITMOVIN_API_KEY });

// Create encoding
const encoding = await bitmovinApi.encoding.encodings.create({
  name: 'My VOD Encoding',
  cloudRegion: BitmovinApi.CloudRegion.AWS_EU_WEST_1,
});

// Add input (S3)
const input = await bitmovinApi.encoding.inputs.s3.create({
  name: 'S3 Input',
  bucketName: 'my-source-bucket',
  accessKey: process.env.AWS_ACCESS_KEY,
  secretKey: process.env.AWS_SECRET_KEY,
});

// Add output (S3)
const output = await bitmovinApi.encoding.outputs.s3.create({
  name: 'S3 Output',
  bucketName: 'my-output-bucket',
  accessKey: process.env.AWS_ACCESS_KEY,
  secretKey: process.env.AWS_SECRET_KEY,
});

// (Continue with codec configurations, streams, muxings, and start encoding)
// Full tutorial: https://developer.bitmovin.com/encoding/docs/nodejs-javascript-sdk
```

**Player:** Bitmovin Player — licensed separately. Runs on every platform listed above.
Particularly valuable for operator/telco deployments where hls.js/Shaka don't reach.

---

### AWS Elemental MediaConvert (VOD) + MediaLive (Live)

**Strengths:**
- Native integration with S3, CloudFront, IAM, Lambda — zero friction if already on AWS
- Pay-per-minute pricing, no contracts
- MediaConvert handles virtually every input format and container
- Good for broadcast-quality workflows (JPEG2000, IMF, MXF inputs)
- MediaLive for live with automatic failover

**Weaknesses:**
- Not developer-friendly by design — heavily console/CloudFormation oriented
- API is verbose and AWS-idiomatic (not REST-idiomatic)
- No managed player; you pair it with your own (hls.js, Shaka, Video.js)
- Smart TV/STB player is your problem to solve

**Best for:** Teams already on AWS who want managed encoding without a new vendor relationship.
Broadcast/post-production workflows with unusual input formats.

**Quickstart (AWS SDK v3):**
```javascript
import { MediaConvertClient, CreateJobCommand } from "@aws-sdk/client-mediaconvert";

const client = new MediaConvertClient({
  region: "us-east-1",
  endpoint: "https://ENDPOINT_ID.mediaconvert.us-east-1.amazonaws.com", // account-specific
});

const jobSettings = {
  Role: "arn:aws:iam::ACCOUNT_ID:role/MediaConvert_Default_Role",
  Settings: {
    Inputs: [{
      FileInput: "s3://my-source-bucket/source.mp4",
      AudioSelectors: { "Audio Selector 1": { DefaultSelection: "DEFAULT" } },
    }],
    OutputGroups: [{
      Name: "Apple HLS",
      OutputGroupSettings: {
        Type: "HLS_GROUP_SETTINGS",
        HlsGroupSettings: {
          Destination: "s3://my-output-bucket/output/",
          SegmentLength: 6,
        },
      },
      Outputs: [
        // 1080p
        {
          VideoDescription: {
            Width: 1920, Height: 1080,
            CodecSettings: { Codec: "H_264", H264Settings: { Bitrate: 4500000, RateControlMode: "CBR" } },
          },
          AudioDescriptions: [{ CodecSettings: { Codec: "AAC", AacSettings: { Bitrate: 192000 } } }],
          ContainerSettings: { Container: "M3U8" },
        },
        // Add more outputs for 720p, 480p, etc.
      ],
    }],
  },
};

const response = await client.send(new CreateJobCommand(jobSettings));
console.log("Job ID:", response.Job?.Id);
```

---

### Mux

**Strengths:**
- Best developer experience in the category — clean REST API, excellent docs
- Simple usage-based pricing (storage + delivery, no encoding fee for basic use)
- Built-in video analytics and real-time monitoring
- Signed URLs and access control out of the box
- Mux Data product is best-in-class for QoE monitoring
- Upload URL → playback URL in < 2 minutes for most videos

**Weaknesses:**
- Less control over encoding parameters (intentionally abstracted)
- No on-premise option
- Smart TV / STB player coverage relies on third parties
- Not the right fit for highly custom encoding workflows

**Best for:** Startups and developer teams who want fast time-to-market. Great for
user-generated content (UGC) platforms, video-first apps, or anywhere QoE analytics matter.

**Quickstart (Node.js):**
```javascript
import Mux from '@mux/mux-node';

const mux = new Mux({
  tokenId: process.env.MUX_TOKEN_ID,
  tokenSecret: process.env.MUX_TOKEN_SECRET,
});

// Create asset from URL
const asset = await mux.video.assets.create({
  input: [{ url: 'https://storage.example.com/source.mp4' }],
  playback_policy: ['public'],
  // Optional: set video quality level (basic / plus / premium)
  // Note: encoding_tier is deprecated — use video_quality instead
  video_quality: 'plus', // 'basic' (free), 'plus', or 'premium'
});

// Get playback URL
const playbackId = asset.playback_ids?.[0]?.id;
const hlsUrl = `https://stream.mux.com/${playbackId}.m3u8`;
console.log('HLS URL:', hlsUrl);
```

**Player:** Mux Player (open source React component), or use hls.js/Shaka directly with the
`.m3u8` URL. Mux Player includes built-in analytics reporting.

```bash
npm install @mux/mux-player
```

```jsx
import MuxPlayer from '@mux/mux-player-react';

<MuxPlayer
  playbackId={playbackId}
  metadata={{ video_title: 'My Video', viewer_user_id: 'user-123' }}
/>
```

---

### Cloudflare Stream

**Strengths:**
- Extremely simple: upload → get playback URL, done
- Pricing is simple (storage + watched minutes)
- Global Cloudflare network for delivery — no separate CDN needed
- Good for small-to-medium scale without ops complexity
- Stream Live for simple live streaming

**Weaknesses:**
- Least control over encoding; black-box
- No custom ABR ladders or advanced codec options
- No native DRM (relies on signed URLs for basic access control)
- Limited analytics vs. Mux

**Best for:** Teams who want the simplest possible video infra and don't need DRM,
custom encoding, or TV platforms. Internal tools, low-complexity consumer apps.

**Quickstart:**
```bash
# Upload via API
curl -X POST \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -F file=@source.mp4 \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/stream"
```

```javascript
// Or via tus resumable upload (recommended for large files)
import * as tus from 'tus-js-client';

const upload = new tus.Upload(file, {
  endpoint: `https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/stream`,
  headers: { Authorization: `Bearer ${CF_API_TOKEN}` },
  metadata: { name: file.name },
  onSuccess: () => console.log('Upload complete'),
});
upload.start();
```

---

### Wowza

**Strengths:**
- Deep live streaming features (low-latency, transcoding, DVR, nDVR)
- Wowza Streaming Engine is deployable on-premise or in your own cloud
- Good for broadcast and telco workflows requiring local data residency
- Long track record in enterprise and broadcast

**Weaknesses:**
- Older API design; less developer-friendly than Mux
- More operationally complex (Java-based server)
- Player is Video.js-based; less competitive on TV/STB vs. Bitmovin

**Best for:** On-premise or private cloud requirements, complex live channel operations,
organisations with existing Wowza relationships.

---

### JW Player / Brightcove

These are full-service **video platform** products (OVP — Online Video Platform), not just
encoding APIs. They bundle a CMS, player, analytics, monetisation (AVOD/SVOD), and CDN.

**When to consider:** You need a complete managed video publishing platform, not just
encoding infrastructure. Common in media companies and publishers.

**When NOT to consider:** You're building a product that embeds video (not publishing video).
For developer use cases, Mux or Bitmovin are better fits.

---

## Vendor Summary Table

| | Bitmovin | AWS Elemental | Mux | Cloudflare Stream | Wowza |
|---|---|---|---|---|---|
| **Dev experience** | Good | Poor | Excellent | Excellent | Fair |
| **Encoding control** | Excellent | Good | Limited | Minimal | Good |
| **Smart TV / STB player** | ✅ Best-in-class | ❌ | ❌ (3rd party) | ❌ | ⚠️ Basic |
| **DRM** | ✅ Multi-DRM | ✅ | ✅ (Widevine+FP) | ❌ | ✅ |
| **On-premise** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Live** | ✅ | ✅ (MediaLive) | ✅ | ✅ (Stream Live) | ✅ |
| **Analytics** | Good | Basic | Excellent | Basic | Basic |
| **Pricing model** | Per-minute + player license | Per-minute | Storage + delivery | Storage + watched min | License / subscription |
| **Best for** | TV/STB, high-quality VOD | AWS-native teams | Dev-first web/mobile | Simplest possible | On-prem live/broadcast |

---

## Hybrid Architectures

You don't have to pick one vendor for everything. Common hybrid patterns:

**Mux encoding + Bitmovin Player**
Use Mux's simple API for encoding and Bitmovin Player for TV/STB playback.
Mux outputs standard HLS/DASH — any player can consume it.

**ffmpeg for VOD + Bitmovin Player for TV**
Self-host encoding for web/mobile renditions using ffmpeg.
License Bitmovin Player only for the TV/STB surface where it adds unique value.

**AWS MediaConvert + CloudFront + hls.js**
Standard AWS stack for teams already committed to the AWS ecosystem.
Add Bitmovin or Mux only when device coverage becomes a requirement.
