# AWS Live Streaming Quick Start Guide: OBS to Amazon IVS

**Version:** v1.1.0  
**Based on full guide:** [`aws-live-streaming-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-guide-obs-ivs-v1.1.0.md)  
**Recommended path:** OBS Studio on Windows 11 → Amazon IVS Low-Latency Streaming → browser playback  
**Best for:** The simplest AWS-native path from OBS to a public browser-viewable live stream.

This quick-start guide is the short version. It gives you the most common path with the least explanation needed to get a working test stream. For deeper AWS explanations, IAM details, billing notes, troubleshooting, and production hardening, use the full guide.

---

## Before you start

You need a Windows 11 PC, OBS Studio, an AWS account, billing access, and permission to create Amazon IVS resources.

For AWS access, use IAM Identity Center or an IAM user/group with appropriate permissions. Do not use the AWS root user for routine work. For detailed AWS identity and permissions setup, see the full guide:

- Full guide: **Step 0: Set up AWS access, IAM, and permissions**
- Full guide: **Appendix M: Detailed IAM setup**

For the first test, use non-sensitive content. A color source, test scene, or harmless webcam view is better than your full desktop.

---

## What this builds

```text
OBS Studio on Windows 11
   ↓ RTMPS
Amazon IVS Low-Latency Streaming channel
   ↓ playback URL
Browser page using the Amazon IVS Web Player
```

Viewers can open the playback URL or player page without logging in. The playback URL may be public, but the ingest URL, stream key, and AWS credentials must stay private.

---

## Cost warning

Streaming to IVS can create AWS charges. Viewer playback can also create charges.

Before a long test:

1. Set up an AWS Budget.
2. Keep the first stream short.
3. Stop OBS when finished.
4. Confirm IVS is no longer receiving a stream.
5. Delete the IVS channel if you do not plan to use it again.

See the full guide’s billing and cleanup sections for details.

---

## Quick-start steps

### 1. Sign in to AWS and choose a Region

1. Open the AWS Management Console.
2. Choose a Region, such as `us-west-2`.
3. Stay in the same Region while creating and checking IVS resources.

**Checkpoint:** The AWS Console Region selector shows your intended Region.

---

### 2. Set up a quick billing safety net

1. Open **Billing and Cost Management**.
2. Open **Budgets**.
3. Create a small cost budget alert.
4. Add your email address.

**Checkpoint:** You have at least one budget alert configured.

---

### 3. Open Amazon IVS

1. In the AWS Console search box, search for **Amazon IVS**.
2. Open the IVS console.
3. Confirm the Region is still correct.

**Checkpoint:** You can see the IVS channel area.

---

### 4. Create an IVS channel

1. Choose **Channels**.
2. Choose **Create channel**.
3. Name the channel something like `home-test-live`.
4. For a low-cost first test, use `BASIC` if it fits your planned resolution and bitrate.
5. Use `STANDARD` if you want the more typical adaptive-streaming behavior.
6. Do not enable recording for the quick test.
7. Do not enable private playback for the quick test unless you specifically want authentication.
8. Create the channel.

Record these values:

| Value | Notes |
|---|---|
| Ingest endpoint | Used by OBS as the server. |
| Stream key | Secret. Do not share it. |
| Playback URL | Used by the browser player. |

**Checkpoint:** You have the ingest endpoint, stream key, and playback URL.

---

### 5. Install and open OBS Studio

1. Download OBS Studio from the official OBS website.
2. Install OBS on Windows 11.
3. Open OBS.

**Checkpoint:** OBS opens successfully.

---

### 6. Create a safe OBS test scene

1. In **Scenes**, click **+**.
2. Name it `AWS IVS Test`.
3. In **Sources**, click **+**.
4. Add a safe source:
   - **Color Source** for the safest test.
   - **Video Capture Device** for a webcam test.
   - **Display Capture** only if nothing private is visible.
5. Mute audio sources you do not want to broadcast.

**Checkpoint:** OBS preview shows only safe content.

---

### 7. Configure OBS video and audio

Open **Settings → Video**:

| Setting | Starter value |
|---|---|
| Output resolution | 1280×720 |
| FPS | 30 |

Open **Settings → Output → Streaming**:

| Setting | Starter value |
|---|---|
| Encoder | Hardware encoder if stable, otherwise x264 |
| Rate control | CBR |
| Video bitrate | 2500–4500 Kbps |
| Audio bitrate | 128–160 Kbps |
| Keyframe interval | 2 seconds |

**Checkpoint:** Your upload speed comfortably exceeds your total stream bitrate.

---

### 8. Configure OBS to stream to IVS

1. Open **Settings → Stream**.
2. Set **Service** to **Custom**.
3. Set **Server** to the IVS ingest endpoint.
   - If the Console gives only a host/path, use the exact AWS-provided format.
   - If needed, prepend `rtmps://`.
4. Set **Stream Key** to the IVS stream key.
5. Save.

**Checkpoint:** OBS has the correct ingest endpoint and stream key. Do not screenshot or share the stream key.

---

### 9. Start streaming

1. Click **Start Streaming** in OBS.
2. Watch the OBS status bar.
3. Confirm there are no repeated reconnects or major dropped-frame warnings.

**Checkpoint:** OBS says it is streaming.

---

### 10. Confirm IVS receives the stream

1. Return to the IVS Console.
2. Open your channel.
3. Look for active stream, stream health, or metrics.

**Checkpoint:** IVS shows that it is receiving the stream.

---

### 11. Watch the stream in a browser

Use the IVS playback URL with the IVS Web Player.

For the fastest test, use the HTML player example from the full guide:

- Full guide: **Step 10: Watch the stream in a browser**
- Full guide: **Public playback setup**

Replace the sample `PLAYBACK_URL` with your IVS playback URL.

**Checkpoint:** You can see and hear the stream in a browser.

---

## Stop here if your goal is only a quick test

If your goal was only to confirm OBS → IVS → browser playback:

1. Click **Stop Streaming** in OBS.
2. Confirm IVS no longer shows an active stream.
3. Check AWS billing/usage later.
4. Delete the IVS channel if you do not plan to use it again.
5. Keep the AWS Budget active.

You are done.

---

## If something fails

| Problem | Most likely fix |
|---|---|
| OBS cannot connect | Re-copy the ingest endpoint and stream key. |
| IVS shows no active stream | Confirm OBS is streaming to the right Region/channel. |
| Browser does not play | Re-copy the playback URL and use the IVS player example. |
| Stream buffers or drops | Lower bitrate or resolution. |
| Unexpected costs | Stop OBS, confirm IVS is inactive, delete unused resources, and check billing. |
| IAM/permission error | See the full guide Step 0 and Appendix M. |

---

## Where to read more

Use the full guide when you need:

- Exact IAM setup steps
- More detailed AWS Console explanations
- CLI examples
- Troubleshooting tables
- Security notes
- Cleanup details
- Production hardening

Full guide: [`aws-live-streaming-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-guide-obs-ivs-v1.1.0.md)
