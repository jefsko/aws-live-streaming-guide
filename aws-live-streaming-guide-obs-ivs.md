# Live Streaming from a Windows 11 PC to AWS with OBS and Amazon IVS

**Last verified:** June 28, 2026  
**Primary recommended path:** OBS Studio on Windows 11 → Amazon IVS Low-Latency Streaming → Amazon IVS Web Player in a browser  
**Primary use case:** A simple, public, browser-viewable live stream from a home PC, with no viewer login required for the first test.

> **Important:** AWS Console labels, pricing, service limits, and supported features can change. This guide links to official AWS and OBS sources near the claims they support and collects the most important links in [Appendix G](#appendix-g-source-links-and-last-verified-notes).

---

## Table of contents

1. [Prerequisites](#prerequisites)
2. [Brief summary](#brief-summary)
3. [What we are building](#what-we-are-building)
4. [Technologies used](#technologies-used)
5. [Why this architecture was chosen](#why-this-architecture-was-chosen)
6. [Brief architecture comparison](#brief-architecture-comparison)
7. [Recommended primary path](#recommended-primary-path)
8. [Pros and cons of the recommended primary path](#pros-and-cons-of-the-recommended-primary-path)
9. [Cost warning and billing overview](#cost-warning-and-billing-overview)
10. [One-page quick-start path](#one-page-quick-start-path)
11. [Step-by-step setup](#step-by-step-setup)
12. [Verification checkpoints](#verification-checkpoints)
13. [Testing the stream](#testing-the-stream)
14. [Public playback setup](#public-playback-setup)
15. [Troubleshooting](#troubleshooting)
16. [Cleanup, shutdown, and avoiding surprise charges](#cleanup-shutdown-and-avoiding-surprise-charges)
17. [Security notes](#security-notes)
18. [Production hardening later](#production-hardening-later)
19. [Glossary](#glossary)
20. [Index](#index)
21. [Appendixes](#appendixes)

---

## Prerequisites

| Prerequisite | Why you need it | How to confirm it is ready |
|---|---|---|
| Windows 11 PC | This is the encoder workstation that will run OBS. | You can sign in, install software, and use a stable network connection. |
| Stable internet connection | Your PC must upload the stream continuously. | Run a speed test. Your upload speed should comfortably exceed your target video bitrate. |
| AWS account | Needed to create Amazon IVS resources. | You can sign in to the AWS Management Console. |
| AWS billing access | Needed to monitor costs and create budgets. | You can open **Billing and Cost Management** and **AWS Budgets**. AWS Budgets supports cost budgets and alerts.[^aws-budgets] |
| AWS permissions | Needed to create IVS channels, view billing, and optionally use IAM/CLI. | Use an administrator account for a lab account, or an IAM role/user with appropriate IVS and billing permissions. Avoid root credentials for daily work.[^iam-best-practices] |
| OBS Studio | The local encoder that captures screen/camera/audio and sends video to AWS. | Download OBS from the official OBS website. OBS supports Windows 10 and 11.[^obs-download] |
| Modern browser | Needed to watch the stream. | Chrome, Edge, Firefox, or Safari are acceptable. For lowest IVS browser latency, use the Amazon IVS Web Player SDK.[^ivs-player-web] |
| Optional AWS CLI v2 | Useful for repeatability and verification. Console instructions are primary. | Install and configure AWS CLI v2 if you want CLI examples.[^aws-cli-getting-started] |
| Optional text editor | Needed if you create a local HTML test player. | VS Code, Notepad, or another editor is fine. |
| Optional webcam/mic/capture card | Needed if you want live camera or external device input. | OBS can see the device as a source. |
| Safe non-sensitive test content | Prevents accidentally broadcasting private information. | Prepare a test scene, color source, local sample video, or non-sensitive camera view. |
| Initial viewer assumption | Keeps the architecture appropriately simple. | This guide assumes a small public test stream or small audience, not a large enterprise broadcast. |
| Region assumption | Examples use `us-west-2` unless noted. | Choose the AWS Region closest to you and supported by IVS. Amazon IVS is regional for control-plane resources.[^ivs-regions] |

**Checkpoint:** You can sign in to AWS, you can install OBS, you know which AWS Region you will use, and you have non-sensitive content ready for testing.

---

## Brief summary

This guide shows how to stream live audio/video from a Windows 11 PC to AWS and let viewers watch in a browser without logging in.

The recommended setup is:

```text
Windows 11 PC
   ↓ OBS Studio using RTMPS
Amazon IVS Low-Latency Streaming channel
   ↓ IVS playback URL
Browser page using Amazon IVS Web Player
```

Amazon IVS is a managed live-video streaming service. AWS describes IVS Low-Latency Streaming as a managed live-video service that delivers video via RTMPS with latency under five seconds.[^ivs-what-is] IVS handles ingest, processing, delivery, and playback infrastructure, so you do not need to build your own RTMP server, HLS packager, CDN distribution, or transcoding stack for the first working version.[^ivs-streaming-config]

For this guide, **publicly viewable** means a viewer can open a playback URL or player page without logging in. It does **not** mean the ingest URL, stream key, AWS credentials, or other secrets are public.

By the end, you should be able to:

- Create an Amazon IVS channel.
- Configure OBS to stream to IVS.
- Watch the stream in a browser.
- Understand the main costs.
- Stop streaming and clean up resources.
- Know when to use IVS versus MediaLive, MediaPackage, CloudFront, MediaConnect, EC2, Lightsail, NGINX RTMP, or non-AWS relays.

---

## What we are building

We are building a simple live-streaming path:

1. OBS captures your screen, webcam, microphone, or other source on Windows 11.
2. OBS sends the encoded stream to Amazon IVS using RTMPS.
3. Amazon IVS receives the stream through an ingest endpoint and private stream key.
4. Amazon IVS provides a playback URL.
5. A browser-based player loads that playback URL and displays the live stream.

For the first test, we will use an IVS channel without recording and without viewer authentication. That keeps the setup simple. Later sections explain when to enable private playback, custom web hosting, monitoring, logging, and production hardening.

---

## Technologies used

| Technology | Role |
|---|---|
| OBS Studio | Captures and encodes video/audio from the Windows 11 PC. |
| Amazon IVS Low-Latency Streaming | Receives the stream, processes it, and provides playback infrastructure. |
| RTMPS | Secure ingest protocol from OBS to IVS. IVS supports encoders using RTMP, RTMPS, or SRT, with RTMPS providing an encrypted TLS stream.[^ivs-streaming-software] |
| Amazon IVS playback URL | The stream URL used by the player. |
| Amazon IVS Web Player SDK | Browser player optimized for IVS low-latency playback.[^ivs-player-web] |
| AWS Budgets | Optional but strongly recommended for cost alerts.[^aws-budgets] |
| AWS CLI | Optional secondary path for repeatable setup and verification. |

---

## Why this architecture was chosen

### Recommendation

Use **Amazon IVS Low-Latency Streaming** as the primary path.

### Known decision assumptions

| Assumption | Value used in this guide |
|---|---|
| Audience size | Small public test stream or small initial audience. |
| Region | Examples use `us-west-2`; choose a region appropriate for your account and location. |
| Latency target | Low latency; expect roughly under five seconds when using the IVS player, based on AWS IVS Low-Latency documentation.[^ivs-what-is] |
| Viewer access | No login required for the first public test. |
| Ingest access | Private ingest URL and stream key. Never publish the stream key. |
| Cost priority | Prefer usage-based and managed services over always-on servers. |
| Complexity tolerance | Beginner-friendly; AWS Console first, CLI second. |
| Production status | Test or small live stream first; hardening later. |

### Decision log

| Option | Decision | Why |
|---|---|---|
| Amazon IVS Low-Latency Streaming | **Recommended** | Purpose-built for managed live streaming, supports OBS-style ingest, provides playback URL/player path, and avoids building MediaLive/MediaPackage/CloudFront or server infrastructure. |
| MediaLive + MediaPackage + CloudFront | Not first choice | More broadcast-grade and flexible, but much more complex and can involve running/idle resource costs. MediaLive channels incur costs while running, even without input/output.[^medialive-pricing] |
| Lightsail/EC2 + NGINX RTMP | Not first choice | Can be cheap and educational, but you operate the server, security, patching, scaling, playback packaging, and bandwidth risk. |
| MediaConnect | Not first choice | AWS describes MediaConnect as live video transport/routing for high-value broadcast workflows, not a simple browser playback service.[^mediaconnect-what-is] |
| Non-AWS relay | Useful comparison only | Often simpler for social streaming, but does not meet the goal of learning AWS-hosted live streaming. |

---

## Brief architecture comparison

| Option | Summary | Relative cost behavior | Complexity | Browser playback approach | Best use case | Recommendation status |
|---|---|---:|---:|---|---|---|
| OBS → Amazon IVS | Managed live streaming with IVS ingest and playback URL. | Usage-based input/output pricing; no self-managed server. | Low | IVS Web Player SDK or supported player path. | Simple public stream from OBS. | **Recommended primary path** |
| OBS → MediaLive → MediaPackage/CloudFront | Broadcast-style AWS media pipeline. | MediaLive running/idle charges plus packaging/CDN charges. | High | HLS/DASH/CMAF via MediaPackage and CloudFront. | Professional OTT/live event workflows. | Advanced alternative |
| OBS → EC2/Lightsail → NGINX RTMP | Self-managed RTMP server. | Always-on instance plus data transfer; cheaper only at low scale and if managed carefully. | Medium/High | Requires HLS packaging/player setup. | Learning RTMP/server operation. | Not recommended first |
| OBS → MediaConnect | Live video transport/routing. | Flow/output-based charges; transport-focused. | High | Needs another service for browser playback. | Broadcast contribution/distribution. | Not recommended for this goal |
| OBS → third-party relay | Stream to Twitch/YouTube/etc. | Often free or simple, but outside AWS. | Low | Platform player. | Public social streaming. | Useful comparison only |

**Why the rest of this guide uses IVS:** IVS gives the shortest practical AWS-native path from OBS to public browser playback while keeping the setup manageable for a novice.

---

## Recommended primary path

Use this path:

```text
OBS Studio on Windows 11
→ Amazon IVS Low-Latency Streaming channel
→ Amazon IVS playback URL
→ Amazon IVS Web Player in a browser
```

For the **quick test**, use an IVS channel without recording and without playback authorization. For lowest cost testing, consider `BASIC` channel type if its limits fit your input; use `STANDARD` if you want adaptive bitrate/transcoding behavior for varied viewer networks. IVS channel type determines allowable resolution and bitrate, and the current IVS channel types are `STANDARD`, `ADVANCED_SD`, `ADVANCED_HD`, and `BASIC`.[^ivs-channel-types]

---

## Pros and cons of the recommended primary path

### Pros

- Fastest AWS-native route to a working live stream.
- Designed for live video streaming rather than generic compute.
- No EC2/Lightsail server to patch or secure.
- Supports OBS through standard encoder workflows.
- Provides a playback URL.
- Low-latency player path is documented by AWS.
- Easier cleanup than a multi-service MediaLive/MediaPackage/CloudFront stack.

### Cons

- Not the cheapest possible option in every scenario.
- Viewer output costs can grow if many people watch.
- Public playback URL can be shared unless you use access control.
- IVS playback custom-domain behavior is limited; AWS notes that IVS playback URLs should not be proxied behind your own domain.[^ivs-create-channel]
- Some advanced broadcast workflows are better served by MediaLive/MediaPackage/CloudFront.
- Audio-only input is not supported for IVS low-latency streaming.[^ivs-streaming-config]

---

## Cost warning and billing overview

Before creating anything, set up at least a small AWS Budget or plan to watch billing closely. AWS Budgets can send alerts when costs approach or exceed a threshold, though budget data is not real-time and is updated periodically.[^aws-budgets]

For IVS Low-Latency Streaming, AWS states that pricing includes separate fees for video input and output; video input fees depend on channel type.[^ivs-costs] Exact prices vary by region, channel type, and date, so check the official IVS pricing page before running a long stream.[^ivs-pricing]

**Cost behavior to understand:**

| Resource/action | Cost risk |
|---|---|
| IVS live video input | Billed while you send video to IVS. |
| IVS video output | Billed based on viewer consumption. |
| Recording to S3 | Optional; adds storage/request costs. Not used in the quick test. |
| CloudFront/MediaPackage/MediaLive | Not used in recommended quick path. If used later, they add separate charges. |
| EC2/Lightsail server | Not used in recommended path. If used, compute and data transfer can continue until stopped/deleted. |

**Checkpoint:** Before streaming for more than a few minutes, you know where to check AWS billing and you understand that both input and viewers can create charges.

---

## One-page quick-start path

This is the shortest safe path to prove that live streaming works.

> This quick-start is concise but not complete. Use the deeper sections if anything fails, if you want to understand costs/security, or if you plan to leave resources in place.

### Quick-start steps

1. **Sign in to AWS.**
   - Go to the AWS Management Console.
   - Choose the Region you want to use, such as `us-west-2`.

2. **Open Amazon IVS.**
   - In the AWS Console search box, search for **Amazon IVS**.
   - Open the IVS console.
   - Make sure the Region selector is correct.

3. **Create a channel.**
   - Go to **Channels**.
   - Choose **Create channel**.
   - Name it something like `home-test-live`.
   - For a low-cost test, consider `BASIC` if the Console offers it and your stream settings fit its limits; otherwise choose `STANDARD`.
   - Keep playback public for the first test unless you specifically want private playback.
   - Do **not** enable recording for the quick test.
   - Create the channel.

4. **Copy the ingest endpoint, stream key, and playback URL.**
   - Treat the stream key as secret. AWS explicitly says the IVS stream key should be treated like a secret because it allows anyone to stream to the channel.[^ivs-create-channel]
   - Record these values in [Appendix H](#appendix-h-my-setup-values-worksheet).

5. **Install OBS.**
   - Download from the official OBS site.
   - Install it on Windows 11.[^obs-download]

6. **Create a safe OBS test scene.**
   - Add a color source, non-sensitive display, or webcam view that does not reveal private data.
   - Add microphone audio only if you are comfortable broadcasting it.

7. **Configure OBS stream settings.**
   - Open **Settings → Stream**.
   - Service: **Custom**.
   - Server: use the IVS ingest endpoint with `rtmps://` in front if needed.
   - Stream Key: paste the IVS stream key.
   - Open **Settings → Output → Streaming** and start conservatively:
     - Video bitrate: 2500–4500 Kbps for a test.
     - Audio bitrate: 128–160 Kbps.
     - Keyframe interval: 2 seconds. AWS IVS encoder guidance highlights keyframe interval, resolution, bitrate, and frame rate as important settings and lists 2 seconds as a key setting in its streaming setup guidance.[^ivs-streaming-software]

8. **Start streaming in OBS.**
   - Click **Start Streaming**.
   - Watch OBS status for dropped frames or connection errors.

9. **View the stream.**
   - Use the IVS playback URL in a player.
   - For the simplest browser test, use the HTML player in [Public playback setup](#public-playback-setup).

10. **Stop streaming.**
    - Click **Stop Streaming** in OBS.
    - Confirm IVS no longer shows an active stream.

### Stop here if your goal is only a quick test

If you only wanted to prove that OBS can stream to AWS and that a browser can play it:

- You have created an IVS channel.
- You have streamed from OBS.
- You have watched playback in a browser.
- You should now stop OBS streaming.
- You should verify AWS is no longer receiving input.
- You should check billing/usage later.
- You may leave the IVS channel for future tests, but delete it if you do not plan to use it again.

Continue only if you want the full explanation, troubleshooting, CLI equivalents, security notes, production-hardening ideas, and cleanup checklist.

---

## Step-by-step setup

### Step 1: Sign in and choose a Region

1. Open the AWS Management Console.
2. Sign in using your IAM Identity Center/user/admin credentials if available.
3. Avoid using the root user for day-to-day work. AWS recommends not using root for everyday tasks and recommends protecting root credentials.[^iam-root]
4. In the top-right Region selector, choose your Region. Examples in this guide use `us-west-2`.

**Expected result:** The Console is open in your chosen Region.

**If it fails:** If you cannot access billing or IVS, your user/role may lack permissions.

**Checkpoint:** The top-right Region selector shows your intended Region.

---

### Step 2: Set up a budget alert

1. Open **Billing and Cost Management**.
2. Open **Budgets**.
3. Create a cost budget.
4. Choose a small threshold that fits your comfort level.
5. Add your email address for alerts.
6. Save the budget.

AWS Budgets can alert when cost approaches or exceeds your threshold, but the data is not instant.[^aws-budgets]

**Expected result:** You have at least one budget alert.

**If it fails:** You may not have billing permissions. Use an account/admin user that can manage billing.

**Checkpoint:** AWS Budgets shows your new budget.

---

### Step 3: Open Amazon IVS

1. In AWS Console search, type **IVS** or **Amazon IVS**.
2. Open the Amazon IVS console.
3. Confirm the Region selector is still correct.

If the AWS Console UI differs from this guide, do not guess. Use the Amazon IVS documentation and the Console search/navigation to find the closest current **Channels** area. AWS documentation has dedicated Console instructions for creating an IVS channel, but Console UI labels can change.[^ivs-create-channel-console]

**Expected result:** You are in the IVS console.

**Checkpoint:** You can see the IVS navigation area, including channels or channel creation.

---

### Step 4: Create an IVS channel

1. Go to **Channels**.
2. Choose **Create channel**.
3. Name the channel: `home-test-live`.
4. Select a channel type:
   - `BASIC`: best for a low-cost first test if your stream fits its limits.
   - `STANDARD`: better if you want IVS-generated adaptive renditions for viewer network variability.
   - `ADVANCED_SD` or `ADVANCED_HD`: consider later if you need the newer advanced transcode presets.
5. Keep latency mode as low-latency/default if the Console offers the option.
6. For quick test playback, do **not** enable playback authorization.
7. Do **not** enable auto-recording for the quick test.
8. Create the channel.

AWS says IVS channel type determines allowable resolution and bitrate; verify current limits in the Channel Types page before selecting final settings.[^ivs-channel-types]

**Expected result:** IVS creates a channel and provides:
- Channel ARN
- Ingest endpoint
- Stream key
- Playback URL

**Checkpoint:** You have the ingest endpoint, stream key, and playback URL recorded.

---

### Step 5: Create a safe OBS test scene

1. Open OBS.
2. In **Scenes**, click **+** and create `AWS IVS Test`.
3. In **Sources**, click **+**.
4. Add one safe source:
   - **Color Source** for a no-risk test, or
   - **Video Capture Device** for a webcam test, or
   - **Display Capture** only if your screen contains no private information.
5. In the **Audio Mixer**, mute anything you do not intend to stream.

OBS is free/open-source software for video recording and live streaming and is available for Windows from the official OBS site.[^obs-download]

**Expected result:** OBS preview shows non-sensitive content.

**Checkpoint:** Nothing private is visible or audible.

---

### Step 6: Configure OBS video/audio

Open **Settings → Video**:

| Setting | Starter value |
|---|---|
| Base canvas resolution | 1280×720 or your monitor resolution |
| Output scaled resolution | 1280×720 for easier testing |
| FPS | 30 |

Open **Settings → Output → Streaming**. If OBS is in Simple mode, switch to Advanced only if needed.

| Setting | Starter value |
|---|---|
| Encoder | Hardware encoder if stable; otherwise x264 |
| Rate control | CBR |
| Video bitrate | 2500–4500 Kbps for 720p/30 test |
| Keyframe interval | 2 seconds |
| Audio bitrate | 128–160 Kbps |

AWS IVS streaming guidance emphasizes encoder settings such as keyframe interval, resolution, bitrate, and frame rate; AWS also notes that audio-only input is not supported for IVS low-latency streaming.[^ivs-streaming-config]

**Expected result:** OBS is configured for a modest, stable test stream.

**Checkpoint:** Your total upload bitrate is comfortably below your available upload speed.

---

### Step 7: Configure OBS stream target

1. Open **Settings → Stream**.
2. Set **Service** to **Custom**.
3. Set **Server** to the IVS ingest endpoint.
   - If the endpoint appears without a protocol, use `rtmps://` plus the ingest endpoint.
   - Example: `rtmps://a1b2c3d4e5f6.global-contribute.live-video.net/app/`
   - Use the exact format shown by the current IVS Console/docs if they differ.
4. Set **Stream Key** to your IVS stream key.
5. Save.

AWS IVS documentation says you can use encoders that support RTMP, RTMPS, or SRT, and the IVS Getting Started instructions include OBS setup.[^ivs-streaming-software]

**Expected result:** OBS is ready to send to IVS.

**Checkpoint:** The stream key is not visible in screenshots or shared documents.

---

### Step 8: Start streaming

1. Click **Start Streaming** in OBS.
2. Watch the OBS status bar.
3. Look for:
   - Green status indicator
   - Stable bitrate
   - Low or no dropped frames
   - No reconnect loop

**Expected result:** OBS sends video to IVS.

**If it fails:**
- Check stream key.
- Check ingest endpoint.
- Check firewall/VPN.
- Lower bitrate.
- Make sure your channel and OBS are in compatible settings.

**Checkpoint:** OBS indicates it is streaming.

---

### Step 9: Confirm IVS is receiving the stream

1. Return to the IVS Console.
2. Open the channel.
3. Look for stream health/status/active stream indicators.
4. If available, view live metrics.

AWS IVS troubleshooting documentation covers unexpected behavior in broadcasting and playback.[^ivs-troubleshooting]

**Expected result:** IVS shows an active stream or current session.

**Checkpoint:** IVS status agrees with OBS: streaming is active.

---

### Step 10: Watch the stream in a browser

Use the IVS playback URL with the IVS Web Player SDK. AWS’s IVS Web Player Getting Started page provides a script-tag setup and sample code using the IVS player CDN.[^ivs-player-getting-started]

Create a file named `ivs-test-player.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Amazon IVS Test Player</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://player.live-video.net/1.53.0/amazon-ivs-player.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 2rem; }
    video { width: 100%; max-width: 960px; background: #000; }
  </style>
</head>
<body>
  <h1>Amazon IVS Test Player</h1>
  <p>Replace PLAYBACK_URL below with your IVS playback URL.</p>
  <video id="video-player" playsinline controls></video>

  <script>
    const playbackUrl = "PLAYBACK_URL";

    if (IVSPlayer.isPlayerSupported) {
      const player = IVSPlayer.create();
      player.attachHTMLVideoElement(document.getElementById("video-player"));
      player.load(playbackUrl);
      player.play();
    } else {
      document.body.insertAdjacentHTML(
        "beforeend",
        "<p>This browser does not support the Amazon IVS player.</p>"
      );
    }
  </script>
</body>
</html>
```

Replace `PLAYBACK_URL` with your IVS playback URL.

**Note:** The script version shown above was current in the AWS IVS Web Player Getting Started page at verification time. Confirm the current version in AWS docs before publishing a long-lived guide or production page.[^ivs-player-getting-started]

**Expected result:** Your live stream plays in the browser.

**Checkpoint:** You can see and hear the live stream.

---

## Verification checkpoints

| Checkpoint | Success condition |
|---|---|
| AWS sign-in | You can access AWS Console and IVS in the chosen Region. |
| Budget | AWS Budgets has a cost budget or alert. |
| IVS channel | Channel exists and has ingest endpoint, stream key, playback URL. |
| OBS scene | OBS preview shows safe content. |
| OBS stream settings | Bitrate, FPS, resolution, and keyframe interval are reasonable. |
| OBS connection | OBS says it is streaming without errors. |
| IVS ingest | IVS shows the stream as active. |
| Playback | Browser player displays the live stream. |
| Audio | Expected audio is present; private audio is not accidentally included. |
| Latency | Delay is understood and acceptable for the chosen path. |
| Security | Stream key remains private. |
| Cost | You know where to check usage and how to stop/delete resources. |
| Cleanup | OBS is stopped and AWS resource state is understood. |

---

## Testing the stream

### Basic test

1. Start OBS streaming.
2. Open your player page.
3. Confirm picture.
4. Confirm audio.
5. Wave at the camera or move a visible window to estimate latency.
6. Stop streaming.
7. Refresh the player after a minute and confirm it is no longer live.

### Quality test

| Test | What to watch |
|---|---|
| 5-minute stream | Dropped frames, OBS reconnects, IVS errors. |
| Lower bandwidth | Playback buffering or quality fallback. |
| Different browser | Compatibility and playback controls. |
| Mobile device | iOS/Android playback behavior. |
| Private/incognito browser | Whether playback truly requires no login. |
| Shared network | Whether others on your home network cause upload instability. |

### Upload bandwidth rule of thumb

Your upload speed should be significantly higher than your total stream bitrate. If OBS sends 4.5 Mbps video + audio overhead, do not assume a 5 Mbps upload connection is enough. Leave headroom.

---

## Public playback setup

### What “public” means here

Public playback means viewers do not need to sign in. It does **not** mean:

- The stream key is public.
- AWS credentials are public.
- The ingest URL is safe to publish.
- The stream is protected from being shared.
- Costs are capped automatically.

The IVS playback URL identifies the endpoint for playback. AWS notes that the playback URL can be used globally and automatically selects the best location from the IVS global content delivery network for the viewer.[^ivs-create-channel]

### Browser player choices

| Player approach | Use it when |
|---|---|
| Amazon IVS Web Player SDK | Recommended for IVS low-latency playback. |
| IVS Video.js integration | Useful if you already use Video.js. |
| Native HLS where supported | Simple but may have higher latency or browser differences. |
| hls.js | Useful for HLS workflows outside IVS or fallback contexts. |
| MediaPackage/CloudFront player | Use only in the advanced MediaLive path. |

### Hosting the player page

For a quick test, open the local HTML file on your PC. For sharing with others, host the page somewhere public, such as:

- Amazon S3 static website hosting plus CloudFront
- Your existing website
- A small static hosting service
- A local web server only for LAN testing

Do not put your stream key in the player page. Only the playback URL belongs in the browser page.

---

## Troubleshooting

| Symptom | Likely cause | How to confirm | Fix | Prevention |
|---|---|---|---|---|
| OBS cannot connect | Wrong ingest URL or stream key | OBS log/status shows connection failure | Re-copy ingest endpoint and stream key from IVS | Use the setup worksheet |
| IVS shows no active stream | OBS not actually sending or wrong Region/channel | Check OBS status and IVS channel Region | Match Region/channel; restart stream | Label resources clearly |
| Stream disconnects immediately | Bitrate/resolution exceeds channel type or unstable upload | OBS reconnects/drops frames | Lower resolution/bitrate; check channel type limits | Start with 720p/30 |
| Browser player does not load | Bad playback URL or player script issue | Browser console errors | Replace playback URL; verify IVS player script version | Use official IVS sample |
| Audio but no video | OBS source/encoder issue | OBS preview or browser shows black video | Check source visibility and encoder settings | Test with color source |
| Video but no audio | Audio source muted/wrong device | OBS mixer meters don’t move | Select correct mic/desktop source | Name audio sources |
| High latency | Player/browser/network/settings | Compare OBS motion to playback | Use IVS player, 2s keyframe, lower buffering | Avoid third-party player unless needed |
| Playback URL shared unexpectedly | Public URL has no auth | Open in private browser | Use private channels/auth later | Treat URL as shareable |
| Unexpected charges | Streaming left running or viewers consumed output | Check billing/usage | Stop OBS; delete unused resources; configure Budgets | Use cleanup checklist |
| IAM permission denied | User lacks IVS/billing permissions | Console or CLI error | Use proper IAM permissions | Least-privilege role with required actions |
| AWS docs don’t match Console | UI changed | Menu labels differ | Use Console search; verify with docs; proceed cautiously | Record Last verified date |
| Region mismatch | Channel created in one Region, looking in another | Channel missing in Console | Switch Region | Write Region in worksheet |

---

## Cleanup, shutdown, and avoiding surprise charges

### Stop the live stream

1. In OBS, click **Stop Streaming**.
2. Wait for OBS to show it is no longer streaming.
3. Open IVS Console.
4. Confirm the stream is no longer active.

Stopping OBS should stop IVS live input for that stream, but you should still confirm the IVS channel is not actively receiving video and check AWS billing/usage later.

### Delete the IVS channel if done

1. Open Amazon IVS.
2. Go to **Channels**.
3. Select your test channel.
4. Delete it if you do not plan to use it again.
5. Confirm deletion.

### Delete optional resources if created

| Resource | What to do |
|---|---|
| IVS channel | Delete if not needed. |
| Recording configuration | Delete if created and not needed. |
| S3 recording bucket | Empty and delete only if you do not need recordings. |
| CloudFront distribution | Disable/delete only if created in advanced path. |
| MediaPackage endpoint/channel | Delete only if created in advanced path. |
| MediaLive channel | Stop/delete if created; note MediaLive has running and idle charge concepts.[^medialive-pricing-doc] |
| EC2/Lightsail instance | Stop or delete if created in alternative path. |
| Logs/metrics | Retain or delete intentionally. |
| Budget | Keep it; it is useful for future AWS experiments. |

### Final cleanup checklist

- [ ] OBS says streaming is stopped.
- [ ] IVS no longer shows an active stream.
- [ ] Playback page no longer shows a live stream.
- [ ] IVS channel deleted or intentionally retained.
- [ ] Recording disabled or recording resources deleted.
- [ ] S3 recordings deleted or intentionally retained.
- [ ] No MediaLive/MediaPackage/CloudFront resources were created unintentionally.
- [ ] AWS Budgets is active.
- [ ] Billing dashboard checked after usage data updates.
- [ ] Stream key was not published in any screenshot, file, or webpage.

---

## Security notes

### Keep private

- AWS account credentials
- IAM access keys
- IVS stream key
- Ingest endpoint if paired with stream key
- Any private playback signing keys or tokens

AWS IVS documentation explicitly says to treat the stream key like a secret because it allows anyone to stream to the channel.[^ivs-create-channel]

### Public playback risk

If playback is public, anyone with the playback URL or embedded player page may be able to watch. If the URL is shared broadly, viewer output costs can grow.

### When to use private playback

Use private playback or playback authorization when:

- The content is sensitive.
- Viewers should be limited to approved people.
- You need per-viewer authorization.
- You want to reduce uncontrolled sharing.
- You are embedding the stream in a paid or private site.

Amazon IVS supports private channels with playback authorization using signed JWTs.[^ivs-private-channels]

### IAM basics

- Do not use root credentials for routine streaming tasks.
- Use MFA.
- Use IAM Identity Center or IAM users/roles with least privilege.
- Do not store access keys in guide files, code, screenshots, or chat logs.
- Do not paste AWS secrets into OBS scene text sources.

---

## Production hardening later

Do not start here. Get the basic stream working first. Revisit this section if the stream becomes recurring, public-facing, business-critical, or sensitive.

| Area | What to improve later |
|---|---|
| Viewer authentication | Use IVS private channels, playback authorization, signed tokens, or an authenticated app. |
| Domain name | Host your player page on your own domain. Note: IVS playback URL custom-domain behavior is limited; do not proxy IVS playback URLs in a way AWS says is unsupported.[^ivs-create-channel] |
| HTTPS/TLS | Serve player pages over HTTPS. |
| Monitoring | Watch IVS metrics, OBS health, viewer errors, and CloudWatch where applicable. |
| Alarms | Create alerts for errors, spend, unexpected stream activity, or service health. |
| Logging | Keep operational notes and logs for failed streams. |
| Cost alerts | Use AWS Budgets and review costs after every test. |
| Scaling | IVS handles delivery scale, but cost and authorization strategy matter. |
| Reliability | Consider redundant encoder/network paths only when needed. |
| Runbook | Document start/stop steps, emergency stop, contact list, and cleanup. |
| Architecture review | Revisit MediaLive/MediaPackage/CloudFront if you need broadcast-grade packaging, multiple outputs, DRM-like workflows, or advanced distribution. |

---

## Glossary

| Term | Meaning |
|---|---|
| OBS | Open Broadcaster Software Studio, local capture/encoding/streaming software. |
| Encoder | Software/hardware that compresses video/audio and sends it to a streaming service. |
| RTMP | Older live ingest protocol often used by encoders. |
| RTMPS | RTMP over TLS; secure ingest protocol supported by IVS. |
| SRT | Secure Reliable Transport, another ingest/transport option in some workflows. |
| HLS | HTTP Live Streaming, common browser/mobile playback format. |
| DASH | Dynamic Adaptive Streaming over HTTP. |
| WebRTC | Low-latency real-time media protocol family, often used for interactive sessions. |
| Latency | Delay between capture and viewer playback. |
| Bitrate | Data rate used by audio/video. Higher can mean better quality but more upload and cost. |
| Frame rate | Frames per second, such as 30 or 60 FPS. |
| Keyframe interval | Time between full video frames; 2 seconds is a common streaming value. |
| Transcoding | Creating new encoded versions/renditions from the input stream. |
| Transmuxing | Repackaging without fully re-encoding. |
| CDN | Content delivery network. |
| IVS | Amazon Interactive Video Service. |
| MediaLive | AWS real-time video service for broadcast/streaming delivery.[^medialive-what-is] |
| MediaPackage | AWS service for packaging live streams into playback formats. |
| CloudFront | AWS CDN often used for HTTP video delivery. |
| MediaConnect | AWS service for reliable live video transport and routing.[^mediaconnect-what-is] |
| Playback URL | URL used by the viewer/player to watch. |
| Ingest URL | URL used by OBS/encoder to send video. |
| Stream key | Secret value that authorizes streaming to a channel. |
| Public access | Viewer can watch without login. |
| Private ingest | Encoder credentials remain secret. |
| Playback authorization | Restricts viewer playback using signed tokens. |

---

## Index

| Keyword | Where to look |
|---|---|
| Amazon IVS | Technologies used; Recommended primary path; Appendix A |
| AWS Budgets | Cost warning; Step 2; Appendix B |
| AWS CLI | Appendix C; Appendix D |
| BASIC channel | Recommended primary path; Step 4 |
| Bitrate | Step 6; Troubleshooting; Worksheet |
| Browser playback | Public playback setup; Step 10 |
| Cleanup | Cleanup section; Appendix J |
| CloudFront | Brief comparison; Appendix A |
| Cost | Cost warning; Billing requirements; Appendix G |
| EC2 | Brief comparison; Appendix A |
| HLS | Public playback; Glossary |
| IAM | Security notes; Appendix B |
| Ingest endpoint | Step 4; Step 7; Worksheet |
| IVS Web Player | Step 10; Public playback setup |
| Keyframe interval | Step 6; Troubleshooting |
| Latency | Known assumptions; Testing; Appendix A |
| Lightsail | Brief comparison; Appendix A |
| MediaConnect | Brief comparison; Appendix A |
| MediaLive | Brief comparison; Appendix A |
| MediaPackage | Brief comparison; Appendix A |
| OBS | Prerequisites; Step 5–8 |
| Playback URL | Step 4; Step 10; Public playback |
| Public playback | Public playback definition; Public playback setup; Security notes |
| Region mismatch | Step 1; Troubleshooting |
| RTMPS | Technologies used; Step 7 |
| Security | Security notes; Production hardening |
| Stream key | Step 4; Step 7; Security notes |
| Troubleshooting | Troubleshooting section; Appendix E |

---

# Appendixes

<details>
<summary><strong>Appendix A: Detailed architecture comparison</strong></summary>

## 1. OBS to Amazon IVS — recommended

- **Overview:** Managed AWS live-streaming service with ingest, processing, delivery, and player support.
- **How it works:** OBS sends RTMPS/SRT/RTMP to IVS; viewers use playback URL/player.
- **Services involved:** Amazon IVS, optional S3 recording, optional CloudWatch/EventBridge.
- **Setup complexity:** Low.
- **Operational complexity:** Low.
- **Cost behavior:** Separate input and output fees; channel type matters.[^ivs-costs]
- **Latency:** Low-latency; AWS describes IVS as delivering via RTMPS with latency under five seconds.[^ivs-what-is]
- **Pros:** Simple, managed, built for streaming, good player path.
- **Cons:** Output costs can grow; not every advanced broadcast feature.
- **Best use:** Simple AWS-hosted public live stream.
- **Recommendation:** Best fit for this guide.

## 2. OBS to MediaLive + MediaPackage/CloudFront — advanced alternative

- **Overview:** Broadcast-grade workflow.
- **How it works:** MediaLive ingests/transcodes; MediaPackage packages; CloudFront distributes.
- **Services involved:** MediaLive, MediaPackage, CloudFront, IAM, CloudWatch, maybe S3.
- **Setup complexity:** High.
- **Operational complexity:** Medium/high.
- **Cost behavior:** MediaLive has running/idle charge concepts; MediaPackage and CloudFront add charges.[^medialive-pricing-doc][^mediapackage-pricing][^cloudfront-pricing]
- **Latency:** Usually higher than IVS low-latency unless carefully designed.
- **Pros:** Flexible, resilient, broadcast-style.
- **Cons:** More expensive/complex for a beginner.
- **Best use:** Professional OTT/live event workflows.
- **Recommendation:** Learn later.

## 3. OBS to Lightsail/EC2 with NGINX RTMP — self-managed alternative

- **Overview:** Build your own RTMP server.
- **How it works:** OBS sends to your server; server serves HLS/RTMP or relays elsewhere.
- **Services involved:** Lightsail or EC2, security groups/firewall, domain/DNS, maybe CloudFront.
- **Setup complexity:** Medium/high.
- **Operational complexity:** High.
- **Cost behavior:** Instance plus data transfer. Lightsail plans include allowances, with overage charges after allowance.[^lightsail-pricing]
- **Latency:** Depends on configuration.
- **Pros:** Educational, flexible, can be cheap for low usage.
- **Cons:** You operate everything.
- **Best use:** Learning server streaming or custom relay.
- **Recommendation:** Not first path.

## 4. OBS to MediaConnect — transport/routing alternative

- **Overview:** Reliable broadcast video transport and routing service.
- **How it works:** Creates flows between sources and destinations.
- **Services involved:** MediaConnect plus downstream encoder/packager/player.
- **Setup complexity:** High.
- **Operational complexity:** High.
- **Cost behavior:** Flow/output pricing; stopped/idle details depend on resource state and pricing docs.[^mediaconnect-pricing]
- **Pros:** Broadcast contribution/distribution.
- **Cons:** Does not itself solve simple public browser playback.
- **Best use:** Professional contribution feeds and partner distribution.
- **Recommendation:** Not for this beginner goal.

## 5. Non-AWS relay

- **Overview:** Use Twitch/YouTube/Vimeo/etc.
- **Pros:** Easiest public viewing.
- **Cons:** Not AWS-hosted, less AWS learning, platform rules/branding.
- **Recommendation:** Mentioned only for context.

</details>

<details>
<summary><strong>Appendix B: AWS services used or referenced</strong></summary>

| Service | Required? | Why mentioned | Cost notes |
|---|---:|---|---|
| Amazon IVS | Yes | Primary streaming service. | Input/output pricing varies by channel type/Region. |
| AWS Budgets | Recommended | Alerts on spend. | Check current AWS cost management terms. |
| IAM | Yes | Access control for AWS account/resources. | No direct cost for normal IAM use. |
| S3 | Optional | IVS recording storage. | Storage/request/data retrieval costs. |
| CloudFront | Optional/alternative | CDN for MediaPackage or player hosting. | Data transfer/requests or flat-rate plans depending configuration. |
| MediaLive | Alternative | Broadcast transcoding/live outputs. | Running and idle resource charges possible. |
| MediaPackage | Alternative | Packaging HLS/DASH/CMAF. | Ingest/origin egress charges. |
| MediaConnect | Alternative | Live video transport/routing. | Flow/output/data pricing. |
| EC2 | Alternative | Self-managed RTMP server. | Instance and data transfer. |
| Lightsail | Alternative | Simplified VPS for RTMP server. | Monthly bundle plus transfer overage. |
| CloudWatch | Optional | Metrics/logging/alarms. | Some metrics/logs/alarms may cost. |

</details>

<details>
<summary><strong>Appendix C: AWS CLI command reference</strong></summary>

> Console instructions are primary. Use CLI only if you are comfortable.

## `aws configure`

Purpose: Configure AWS CLI credentials and default Region.

```powershell
aws configure
```

Required values:
- AWS Access Key ID
- AWS Secret Access Key
- Default Region, such as `us-west-2`
- Output format, such as `json`

AWS recommends using IAM credentials and not root credentials for CLI access.[^aws-cli-getting-started]

## `aws ivs create-channel`

Purpose: Create an IVS channel and stream key.

```powershell
aws ivs create-channel --name "home-test-live" --latency-mode LOW --type BASIC --no-authorized --no-insecure-ingest --region us-west-2
```

Important output:
- `channel.ingestEndpoint`
- `channel.playbackUrl`
- `streamKey.value`

AWS CLI documentation says `create-channel` creates a new IVS channel and associated stream key.[^ivs-cli-create-channel]

## `aws ivs get-channel`

Purpose: Show channel configuration.

```powershell
aws ivs get-channel --arn "CHANNEL_ARN" --region us-west-2
```

## `aws ivs list-stream-keys`

Purpose: List stream keys for a channel.

```powershell
aws ivs list-stream-keys --channel-arn "CHANNEL_ARN" --region us-west-2
```

## `aws ivs delete-channel`

Purpose: Delete an IVS channel.

```powershell
aws ivs delete-channel --arn "CHANNEL_ARN" --region us-west-2
```

Use only after stopping OBS and confirming you no longer need the channel.

</details>

<details>
<summary><strong>Appendix D: AWS CLI task sequences</strong></summary>

## Task: Create a quick-test IVS channel

```powershell
aws ivs create-channel --name "home-test-live" --latency-mode LOW --type BASIC --no-authorized --no-insecure-ingest --region us-west-2
```

Record:
- `channel.arn`
- `channel.ingestEndpoint`
- `channel.playbackUrl`
- `streamKey.value`

## Task: Verify channel

```powershell
aws ivs get-channel --arn "CHANNEL_ARN" --region us-west-2
```

## Task: Delete channel

```powershell
aws ivs delete-channel --arn "CHANNEL_ARN" --region us-west-2
```

</details>

<details>
<summary><strong>Appendix E: Troubleshooting quick reference</strong></summary>

- **OBS reconnect loop:** lower bitrate, verify endpoint/key, disable VPN temporarily.
- **Black video:** check OBS source visibility and display capture permissions.
- **No audio:** check OBS mixer, Windows input device, mute states.
- **High latency:** use IVS player, 2s keyframe interval, avoid third-party native playback where low latency matters.
- **Public playback concern:** use IVS private channels/playback authorization later.
- **Billing surprise:** stop OBS, confirm inactive stream, delete unused resources, check Budgets.

</details>

<details>
<summary><strong>Appendix F: Glossary expansion</strong></summary>

See [Glossary](#glossary). Add new terms to this appendix as the guide evolves.

</details>

<details>
<summary><strong>Appendix G: Source links and last-verified notes</strong></summary>

**Last verified:** June 28, 2026.

Official sources used:

[^ivs-what-is]: Amazon IVS Low-Latency Streaming overview: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/what-is.html  
[^ivs-streaming-config]: Amazon IVS Streaming Configuration: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/streaming-config.html  
[^ivs-streaming-software]: Amazon IVS Getting Started, Set Up Streaming Software: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/getting-started-set-up-streaming.html  
[^ivs-create-channel]: Amazon IVS Getting Started, Create a Channel: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/getting-started-create-channel.html  
[^ivs-create-channel-console]: Amazon IVS Console Instructions for Creating a Channel: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/create-channel-console.html  
[^ivs-player-web]: Amazon IVS Web Player SDK guide: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/player-web.html  
[^ivs-player-getting-started]: Getting Started with the IVS Web Player SDK: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/web-getting-started.html  
[^ivs-costs]: IVS Costs: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/costs.html  
[^ivs-pricing]: Amazon IVS Pricing: https://aws.amazon.com/ivs/pricing/  
[^ivs-channel-types]: IVS Channel Types: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/channel-types.html  
[^ivs-private-channels]: IVS Private Channels: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/private-channels.html  
[^ivs-troubleshooting]: IVS Troubleshooting: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/troubleshooting-faqs.html  
[^ivs-regions]: IVS region/control-plane notes: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/what-is.html  
[^ivs-cli-create-channel]: AWS CLI `ivs create-channel`: https://docs.aws.amazon.com/cli/latest/reference/ivs/create-channel.html  
[^obs-download]: OBS Studio download page: https://obsproject.com/download  
[^aws-cli-getting-started]: AWS CLI getting started: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html  
[^iam-best-practices]: IAM security best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html  
[^iam-root]: Root user best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html  
[^aws-budgets]: Managing costs with AWS Budgets: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html  
[^medialive-what-is]: What is MediaLive: https://docs.aws.amazon.com/medialive/latest/ug/what-is.html  
[^medialive-pricing]: MediaLive pricing page: https://aws.amazon.com/medialive/pricing/  
[^medialive-pricing-doc]: Pricing in MediaLive docs: https://docs.aws.amazon.com/medialive/latest/ug/pricing.html  
[^mediapackage-pricing]: MediaPackage pricing: https://aws.amazon.com/mediapackage/pricing/  
[^cloudfront-pricing]: CloudFront pricing: https://aws.amazon.com/cloudfront/pricing/  
[^cloudfront-live]: CloudFront live streaming docs: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/live-streaming.html  
[^mediaconnect-what-is]: What is MediaConnect: https://docs.aws.amazon.com/mediaconnect/latest/ug/what-is.html  
[^mediaconnect-pricing]: MediaConnect pricing: https://aws.amazon.com/mediaconnect/pricing/  
[^lightsail-pricing]: Lightsail pricing: https://aws.amazon.com/lightsail/pricing/  
[^ec2-pricing]: EC2 On-Demand pricing: https://aws.amazon.com/ec2/pricing/on-demand/  

Known likely-to-change areas:

- AWS Console labels and navigation.
- IVS player SDK version.
- AWS pricing.
- Channel type limits.
- Region availability.
- Playback authorization and restriction options.

</details>

<details>
<summary><strong>Appendix H: My setup values worksheet</strong></summary>

| Field | Value |
|---|---|
| AWS Region |  |
| Recommended architecture selected | Amazon IVS Low-Latency Streaming |
| IVS channel name |  |
| IVS channel ARN |  |
| Channel type |  |
| Ingest endpoint |  |
| Stream key | **Do not paste into shared docs** |
| Playback URL |  |
| Test player file | `ivs-test-player.html` |
| OBS scene name |  |
| Resolution |  |
| Frame rate |  |
| Video bitrate |  |
| Audio bitrate |  |
| Keyframe interval | 2 seconds |
| Expected latency |  |
| Test start time |  |
| Test stop time |  |
| Resources to stop | OBS stream |
| Resources to delete | IVS channel if done |

</details>

<details>
<summary><strong>Appendix I: Decision log</strong></summary>

| Decision | Reason |
|---|---|
| Use IVS as primary path | Simpler than MediaLive stack and purpose-built for managed live streaming. |
| Use OBS | Free, common, Windows 11-compatible, supports custom streaming. |
| Use RTMPS | Secure ingest option supported by IVS. |
| Do not use recording in quick start | Avoids S3 setup and storage costs. |
| Do not enable viewer auth in quick start | Satisfies no-login public playback goal. |
| Mention private channels later | Needed for sensitive or controlled streams. |
| Use Console first | Beginner-friendly. |
| Include CLI second | Useful for repeatability. |

</details>

<details>
<summary><strong>Appendix J: Final cleanup checklist</strong></summary>

- [ ] OBS stopped.
- [ ] IVS stream inactive.
- [ ] IVS channel deleted or intentionally retained.
- [ ] Playback page removed if no longer needed.
- [ ] Stream key not exposed.
- [ ] Recording resources deleted if created.
- [ ] S3 bucket emptied/deleted if created and no longer needed.
- [ ] CloudFront/MediaPackage/MediaLive resources checked if used.
- [ ] EC2/Lightsail resources checked if used.
- [ ] Billing dashboard reviewed after usage updates.
- [ ] AWS Budget remains active.

</details>

<details>
<summary><strong>Appendix K: Screenshot and visual aid suggestions</strong></summary>

| Visual | Where it belongs | Why it helps |
|---|---|---|
| Architecture diagram | Brief summary | Shows PC → IVS → browser flow. |
| AWS Region selector | Step 1 | Prevents Region mismatch. |
| IVS channel creation page | Step 4 | Helps with Console navigation. |
| IVS channel details with sensitive values redacted | Step 4 | Shows where ingest/playback values are. |
| OBS scene setup | Step 5 | Helps beginners understand scenes/sources. |
| OBS Stream settings | Step 7 | Shows where Server and Stream Key go. |
| Browser player page | Step 10 | Confirms what success looks like. |
| Cleanup checklist screenshot | Cleanup section | Reinforces cost control. |

</details>

<details>
<summary><strong>Appendix L: Expandable appendix guidance</strong></summary>

This guide uses expandable `<details>` sections for appendixes. Essential setup instructions remain in the main guide. Appendixes provide deeper reference material, extended comparisons, and checklists.

</details>
