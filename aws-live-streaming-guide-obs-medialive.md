# Live Streaming from a Windows 11 PC to AWS with OBS and AWS Elemental MediaLive

**Last verified:** June 28, 2026  
**Primary recommended path for this version:** OBS Studio on Windows 11 → AWS Elemental MediaLive RTMP Push input → HLS output to AWS Elemental MediaPackage v2 → Amazon CloudFront → browser playback  
**Difficulty level:** Intermediate to advanced. This version is intentionally more advanced than the Amazon IVS version.  
**Quick-start requirement:** Includes a one-page quick-start path before the deeper MediaLive explanation.  
**Primary use case:** A more broadcast-style AWS live-streaming workflow where MediaLive is the central encoder/transcoder and the stream is publicly playable in a browser.

> **Important:** This MediaLive version is more complex and usually more expensive than an Amazon IVS-first workflow. It is useful if you specifically want to learn or use the AWS broadcast-media pipeline: MediaLive for live encoding/transcoding, MediaPackage for live packaging/origin, and CloudFront for public delivery.

> **Current-source note:** AWS Console labels, pricing, service limits, supported protocols, and recommended workflows can change. This guide links to official AWS and OBS sources near the claims they support and collects the most important links in [Appendix G](#appendix-g-source-links-and-last-verified-notes).

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
| Stable internet connection | Your PC must upload the stream continuously to MediaLive. | Run a speed test. Your upload speed should comfortably exceed your target OBS bitrate. |
| AWS account | Needed to create MediaLive, MediaPackage, CloudFront, IAM, and optional monitoring resources. | You can sign in to the AWS Management Console. |
| AWS billing access | Needed to monitor costs and create budgets. | You can open **Billing and Cost Management** and **AWS Budgets**. AWS Budgets supports cost budgets and alerts.[^aws-budgets] |
| AWS permissions | Needed to create MediaLive inputs/channels, MediaPackage v2 resources, CloudFront distributions, IAM roles, and budgets. | Use an admin-capable lab account or an IAM role/user with appropriate permissions. Avoid root credentials for daily work.[^iam-best-practices] |
| OBS Studio | The local encoder that captures screen/camera/audio and pushes RTMP to MediaLive. | Download OBS from the official OBS website. OBS supports Windows 10 and 11.[^obs-download] |
| Modern browser | Needed to watch the HLS stream. | Chrome, Edge, Firefox, or Safari are acceptable. Safari can play HLS natively; other browsers usually need a JavaScript player such as hls.js. |
| Optional AWS CLI v2 | Useful for repeatability and verification. Console instructions are primary. | Install and configure AWS CLI v2 if you want CLI examples.[^aws-cli-getting-started] |
| Optional text editor | Needed if you create a local HTML HLS test player. | VS Code, Notepad, or another editor is fine. |
| Optional webcam/mic/capture card | Needed if you want live camera or external device input. | OBS can see the device as a source. |
| Safe non-sensitive test content | Prevents accidentally broadcasting private information. | Prepare a test scene, color source, local sample video, or non-sensitive camera view. |
| Initial viewer assumption | Keeps the architecture appropriately scoped. | This guide assumes a small public test stream or small audience, not a large enterprise broadcast. |
| Region assumption | Examples use `us-west-2` unless noted. | Choose an AWS Region that supports MediaLive and MediaPackage v2. Check current regional availability before starting.[^aws-regional-services] |

**Checkpoint:** You can sign in to AWS, you can install OBS, you know which AWS Region you will use, and you have non-sensitive content ready for testing.

---

## Brief summary

This guide shows how to stream live audio/video from a Windows 11 PC to AWS using **OBS Studio and AWS Elemental MediaLive**, then make the stream publicly viewable in a browser.

The recommended setup for this version is:

```text
Windows 11 PC
   ↓ OBS Studio using RTMP Push
AWS Elemental MediaLive input and channel
   ↓ HLS output over HTTPS
AWS Elemental MediaPackage v2 origin endpoint
   ↓ CloudFront distribution
Browser playback using HLS
```

This is **more advanced** than the simpler Amazon IVS approach. MediaLive is a real-time video service for creating live outputs for broadcast and streaming delivery.[^medialive-what-is] In this architecture, MediaLive receives an RTMP Push stream from OBS, encodes/transcodes it, sends HLS output to MediaPackage, and CloudFront serves the packaged stream to public viewers. AWS MediaLive documentation describes setting up an RTMP Push input for an upstream system delivering source content from the public internet.[^medialive-rtmp-push] AWS also documents sending an HLS output group from MediaLive to MediaPackage over HTTPS.[^medialive-hls-mediapackage] CloudFront documentation describes serving live video formatted with MediaPackage through a CloudFront distribution.[^cloudfront-live-mediapackage]

For this guide, **publicly viewable** means a viewer can open a playback URL or player page without logging in. It does **not** mean the MediaLive input, OBS stream details, AWS credentials, MediaPackage origin, CloudFront configuration, or any secrets should be public.

By the end, you should be able to:

- Create a MediaLive RTMP Push input.
- Create a MediaLive channel.
- Configure OBS to push video into MediaLive.
- Send MediaLive output to MediaPackage v2.
- Serve the live stream through CloudFront.
- Watch the stream in a browser.
- Understand why this is more advanced and more cost-sensitive than IVS.
- Stop streaming and clean up resources.

---

## What we are building

We are building a broadcast-style AWS live-streaming chain:

1. OBS captures your screen, webcam, microphone, or other source on Windows 11.
2. OBS pushes an RTMP stream to a MediaLive RTMP Push input.
3. MediaLive receives the stream and encodes it into one or more HLS renditions.
4. MediaLive sends the HLS output to MediaPackage v2.
5. MediaPackage v2 acts as the origin/packager for viewer playback.
6. CloudFront distributes the live stream publicly.
7. A browser loads the CloudFront HLS URL through a player.

For the first test, we will keep the channel simple: one input, one pipeline where practical, a basic HLS output, public playback, no DRM, no recording, no ad insertion, and no advanced redundancy unless you choose to add it later.

---

## Technologies used

| Technology | Role |
|---|---|
| OBS Studio | Captures and encodes video/audio from the Windows 11 PC. |
| RTMP Push | Protocol OBS uses to push the live source to MediaLive. MediaLive documentation explicitly supports RTMP Push inputs from upstream systems.[^medialive-rtmp-push] |
| AWS Elemental MediaLive | Receives, encodes, and outputs the live stream. |
| AWS Elemental MediaPackage v2 | Packages/originates the live HLS stream for playback. MediaPackage supports HLS and Low-Latency HLS for delivery to devices and browsers.[^mediapackage-hls-llhls] |
| Amazon CloudFront | CDN that serves the MediaPackage-originated live stream to viewers. CloudFront documentation includes a workflow for serving live video formatted with MediaPackage.[^cloudfront-live-mediapackage] |
| HLS | HTTP Live Streaming playback format. |
| hls.js or native HLS playback | Browser playback path. Safari supports HLS natively; many other browsers use hls.js. |
| IAM | Permissions and service roles. |
| AWS Budgets | Optional but strongly recommended for cost alerts.[^aws-budgets] |
| AWS CLI | Optional secondary path for repeatable setup and verification. |

---

## Why this architecture was chosen

### Recommendation for this version

Use **OBS → AWS Elemental MediaLive → AWS Elemental MediaPackage v2 → Amazon CloudFront**.

This is the recommended method for this specific MediaLive-focused version of the guide, not because it is the simplest or cheapest path, but because you asked for a guide that uses **AWS Elemental MediaLive and OBS as the recommended method**.

### Known decision assumptions

| Assumption | Value used in this guide |
|---|---|
| Audience size | Small public test stream or small initial audience. |
| Region | Examples use `us-west-2`; verify MediaLive and MediaPackage v2 availability in your Region. |
| Latency target | Standard HLS latency unless you later tune for LL-HLS. MediaPackage supports HLS and LL-HLS, but low-latency tuning adds complexity.[^mediapackage-hls-llhls] |
| Viewer access | No login required for the first public test. |
| Ingest access | MediaLive input security group restricts source IP where practical; OBS RTMP destination details remain private. |
| Cost priority | Cost-aware, but this workflow is not the cheapest AWS-native path. |
| Complexity tolerance | Intermediate/advanced. This is substantially more complex than IVS. |
| Test-stream versus production-stream assumption | Start with a test stream; harden later. |

### Decision log

| Option | Decision | Why |
|---|---|---|
| MediaLive + MediaPackage v2 + CloudFront | **Recommended for this version** | Uses AWS Elemental MediaLive as requested and creates a complete browser-playback workflow. |
| Amazon IVS | Not primary in this version | IVS is usually simpler for OBS-to-browser streaming, but this version is intentionally MediaLive-first. |
| MediaLive direct to S3/other HLS origin | Not primary | Possible in some workflows, but MediaPackage + CloudFront is the more AWS media-services-oriented origin/CDN pattern. |
| Lightsail/EC2 + NGINX RTMP | Not primary | Lower-level self-managed server path; does not teach MediaLive. |
| MediaConnect | Not primary | MediaConnect is transport/routing-oriented and does not by itself create public browser playback. |
| Non-AWS relay | Useful comparison only | Does not satisfy the AWS MediaLive learning goal. |

---

## Brief architecture comparison

| Option | Summary | Relative cost behavior | Complexity | Browser playback approach | Best use case | Recommendation status |
|---|---|---:|---:|---|---|---|
| OBS → MediaLive → MediaPackage v2 → CloudFront | Broadcast-style AWS live pipeline. | Higher; MediaLive input/output charges plus MediaPackage/CloudFront usage. | High | HLS/LL-HLS via CloudFront and browser player. | Learning/using AWS broadcast-media workflow. | **Recommended for this version** |
| OBS → Amazon IVS | Managed live streaming with IVS ingest and playback URL. | Often simpler usage-based model. | Low | IVS Web Player SDK or IVS playback URL. | Simple public stream from OBS. | Simpler alternative |
| OBS → EC2/Lightsail → NGINX RTMP | Self-managed RTMP server. | Instance plus data transfer; cheaper only if managed carefully. | Medium/High | Requires HLS/player setup. | Learning RTMP/server operation. | Not recommended first |
| OBS → MediaConnect | Live video transport/routing. | Flow/output-based costs. | High | Needs another service for browser playback. | Broadcast contribution/distribution. | Not recommended for this goal |
| OBS → third-party relay | Stream to Twitch/YouTube/etc. | Often free or platform-priced. | Low | Platform player. | Public social streaming. | Useful comparison only |

**Why the rest of this guide uses MediaLive:** This version is meant to teach the AWS Elemental MediaLive workflow. It is the correct version if your goal is to learn or operate MediaLive, but not if your top priority is the simplest or cheapest OBS-to-browser AWS stream.

---

## Recommended primary path

Use this path:

```text
OBS Studio on Windows 11
→ MediaLive RTMP Push input
→ MediaLive channel with HLS output group
→ MediaPackage v2 channel/origin endpoint
→ CloudFront distribution
→ Browser playback with HLS player
```

### Why MediaPackage and CloudFront are included

MediaLive is the live encoder/transcoder. It is not, by itself, the public browser playback page. For public browser playback, the MediaLive output needs an origin and delivery layer. In this guide, MediaPackage v2 is the origin/packager, and CloudFront is the CDN. AWS CloudFront documentation describes creating a CloudFront distribution for live video formatted with MediaPackage.[^cloudfront-live-mediapackage]

---

## Pros and cons of the recommended primary path

### Pros

- Uses AWS Elemental MediaLive as requested.
- Teaches a real AWS broadcast-media architecture.
- More flexible than IVS for complex output ladders and downstream packaging.
- Works with HLS browser playback.
- Can evolve toward production workflows with CloudFront, monitoring, alarms, and access control.
- MediaPackage can support HLS and LL-HLS workflows.[^mediapackage-hls-llhls]

### Cons

- More complex than IVS.
- More expensive and easier to misconfigure.
- More resources to create and clean up.
- MediaLive has running and idle cost behavior; stopped channels and idle push inputs can still have charges.[^medialive-pricing-doc][^medialive-pricing-page]
- MediaLive channel startup is not instant. AWS documentation says most AWS Cloud channels start in 3 minutes or less, but up to 10 minutes can be normal.[^medialive-start-stop]
- Public playback needs more than MediaLive alone.
- Console workflows are detailed and can change.

---

## Cost warning and billing overview

This workflow can produce charges from multiple services:

| Service/resource | Cost behavior |
|---|---|
| MediaLive channel | Running input/output charges; stopped/idle channel charges may apply.[^medialive-pricing-doc] |
| MediaLive RTMP Push input | Push inputs can incur idle resource charges when not associated with a running channel or attached to a stopped channel.[^medialive-pricing-page] |
| MediaPackage v2 | Packaging/origin charges, requests, and egress depending on current pricing. |
| CloudFront | Data transfer and request charges; viewer traffic can grow costs. |
| CloudWatch | Metrics/logs/alarms may have charges. |
| AWS Budgets | Useful for alerts; confirm current pricing/limits. |
| Data transfer | Viewer delivery and cross-service/Internet delivery can add cost depending on path and Region. |

AWS MediaLive pricing documentation states that running channels have input and output charges, and that the running input charge applies to inputs attached to a running channel even if they are not receiving content.[^medialive-pricing-doc] The MediaLive pricing page also states that idle resources include push inputs and channels not in use, with idle resource pricing details on that page.[^medialive-pricing-page]

**Checkpoint:** Before creating MediaLive resources, you understand that stopping OBS is not enough. You must stop/delete AWS-side resources as appropriate.

---

## One-page quick-start path

This is the shortest safe path to prove the MediaLive workflow works. It is still more advanced than IVS.

> This quick-start is concise but not complete. Use the deeper sections if anything fails, if you want to understand costs/security, or if you plan to leave resources in place.

### Quick-start steps

1. **Sign in to AWS.**
   - Use the AWS Management Console.
   - Choose the Region you want to use, such as `us-west-2`.

2. **Set up a cost budget.**
   - Open **Billing and Cost Management → Budgets**.
   - Create a small budget alert before starting.

3. **Create the downstream origin first: MediaPackage v2.**
   - Open **AWS Elemental MediaPackage**.
   - Create a live channel/channel group/origin endpoint according to the current MediaPackage v2 Console flow.
   - Choose HLS for the first browser-playback test.
   - Record the MediaPackage input/destination information needed by MediaLive.
   - If the Console differs from AWS docs, proceed cautiously and verify against current official documentation.

4. **Create a CloudFront distribution for MediaPackage playback.**
   - Open **CloudFront**.
   - Create a distribution using the MediaPackage endpoint as the origin.
   - Use HTTPS.
   - Record the CloudFront domain name.
   - CloudFront documentation describes serving live video formatted with MediaPackage through a distribution.[^cloudfront-live-mediapackage]

5. **Create a MediaLive input security group.**
   - Open **AWS Elemental MediaLive**.
   - Create an input security group that allows your public IP address.
   - Avoid allowing `0.0.0.0/0` except for a temporary lab test, and remove it immediately afterward.

6. **Create a MediaLive RTMP Push input.**
   - In MediaLive, create an **RTMP Push** input.
   - Attach the input security group.
   - Record the destination URL(s). AWS documents RTMP Push as the mode where the upstream system pushes content to MediaLive.[^medialive-rtmp-push]

7. **Create a MediaLive channel.**
   - Attach the RTMP Push input.
   - Choose a simple HLS output group.
   - Send the HLS output to MediaPackage over HTTPS. AWS documents HLS output groups to MediaPackage.[^medialive-hls-mediapackage]
   - Use a simple single-rendition or small adaptive ladder for the first test.

8. **Configure OBS.**
   - Create a safe test scene.
   - Configure **Settings → Stream → Custom**.
   - Server: MediaLive RTMP destination URL.
   - Stream key/path: the MediaLive application/stream name required by the destination.
   - Set bitrate and keyframe interval conservatively.

9. **Start the MediaLive channel.**
   - Start the channel manually. AWS says MediaLive channels do not start automatically except recovery cases, and most AWS Cloud channels start in 3 minutes or less, though up to 10 minutes can be normal.[^medialive-start-stop]

10. **Start OBS streaming.**
    - Confirm OBS is pushing video.
    - Confirm MediaLive input is receiving content.

11. **Watch playback.**
    - Use the CloudFront HLS URL in a browser/player.
    - Safari may play HLS natively; Chrome/Edge/Firefox commonly need hls.js.

12. **Stop everything.**
    - Stop OBS.
    - Stop the MediaLive channel.
    - Delete or intentionally retain test resources.
    - Check usage/billing later.

### Stop here if your goal is only a quick test

If you only wanted to prove OBS → MediaLive → MediaPackage → CloudFront playback:

- You have created a complete MediaLive-based live pipeline.
- You have streamed from OBS.
- You have watched playback in a browser.
- You should now stop OBS.
- You should stop the MediaLive channel.
- You should delete unused MediaLive inputs/channels if you do not need them.
- You should delete unused MediaPackage and CloudFront resources if this was only a test.
- You should check billing/usage after AWS cost data updates.

Continue only if you want the full explanation, troubleshooting, CLI equivalents, security notes, production-hardening ideas, and cleanup checklist.

---

## Step-by-step setup

### Step 1: Sign in and choose a Region

1. Open the AWS Management Console.
2. Sign in using IAM Identity Center, an IAM user, or an administrator-capable lab role.
3. Avoid using the root user for routine work. AWS recommends not using root credentials for everyday tasks.[^iam-root]
4. In the top-right Region selector, choose your Region. Examples use `us-west-2`.

**Expected result:** The Console is open in your chosen Region.

**If it fails:** If you cannot access MediaLive, MediaPackage, CloudFront, IAM, or Billing, your user/role may lack permissions.

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

**If it fails:** You may not have billing permissions.

**Checkpoint:** AWS Budgets shows your new budget.

---

### Step 3: Create or identify the MediaLive service role

MediaLive needs permission to write outputs to downstream services and access resources. The MediaLive Console may offer to create or use a service role, often named something like `MediaLiveAccessRole`.

1. Open **AWS Elemental MediaLive**.
2. Look for the channel creation flow or role setup prompt.
3. If prompted, allow MediaLive to create/use the service role.
4. If your organization uses restricted IAM, ask your AWS administrator for the approved MediaLive service role.

**AWS Console mismatch caution:** AWS Console role setup screens can change. If the Console does not match this guide, follow the current MediaLive documentation and your account’s IAM policy requirements. Do not create overly broad roles in a production account.

**Expected result:** MediaLive has a valid role it can use.

**Checkpoint:** Channel creation does not fail with an IAM role error.

---

### Step 4: Create the MediaPackage v2 destination

Create the downstream origin before the MediaLive channel so you have the destination URLs/credentials MediaLive needs.

1. Open **AWS Elemental MediaPackage**.
2. Use the current MediaPackage v2 Console workflow.
3. Create a channel group if required by the current Console.
4. Create a channel.
5. Create an origin endpoint for HLS.
6. Record:
   - Channel group name
   - Channel name
   - Origin endpoint name
   - HLS playback/origin URL
   - Any ingest/destination details required by MediaLive

MediaPackage v2 supports HLS and LL-HLS for delivery to devices and browsers.[^mediapackage-hls-llhls]

**Expected result:** MediaPackage v2 has a live channel/origin endpoint ready for MediaLive output.

**If it fails:** Verify Region support, permissions, and whether you are in MediaPackage v2 versus older MediaPackage UI.

**Checkpoint:** You have the MediaPackage endpoint information needed by MediaLive.

---

### Step 5: Create a CloudFront distribution

1. Open **CloudFront**.
2. Choose **Create distribution**.
3. Set the origin to the MediaPackage endpoint/origin domain.
4. Use HTTPS.
5. Configure cache behavior according to the MediaPackage/CloudFront live-streaming documentation.
6. Create the distribution.
7. Wait for deployment.
8. Record the CloudFront distribution domain name.

CloudFront documentation says that if you formatted a live stream using MediaPackage, you can create a CloudFront distribution and configure cache behaviors to serve the live stream.[^cloudfront-live-mediapackage]

**Expected result:** CloudFront has a distribution pointing to the MediaPackage origin.

**If it fails:** Verify the origin URL, protocol, and MediaPackage endpoint status.

**Checkpoint:** CloudFront status is deployed or in progress, and you know the distribution domain name.

---

### Step 6: Create a MediaLive input security group

A MediaLive input security group controls which source IPs can push content into the input.

1. Determine your public IPv4 address.
2. Open **AWS Elemental MediaLive**.
3. Go to **Input security groups**.
4. Create an input security group.
5. Add your public IP as a `/32` CIDR.
   - Example: `203.0.113.25/32`
6. Save.

**Avoid:** `0.0.0.0/0` unless you are doing a temporary lab test and will immediately remove it.

**Expected result:** MediaLive will accept RTMP pushes only from your allowed IP.

**Checkpoint:** Input security group exists and contains your public IP CIDR.

---

### Step 7: Create a MediaLive RTMP Push input

1. In MediaLive, go to **Inputs**.
2. Choose **Create input**.
3. Choose **RTMP Push**.
4. Name it `obs-rtmp-push-input`.
5. Attach the input security group.
6. Create the input.
7. Record the destination URL(s) that MediaLive provides.

AWS MediaLive documentation says RTMP Push is the workflow where the upstream system pushes content to MediaLive.[^medialive-rtmp-push]

**Expected result:** MediaLive has an RTMP Push input for OBS.

**Checkpoint:** You have the input destination URL(s) for OBS.

---

### Step 8: Create a MediaLive channel

1. In MediaLive, go to **Channels**.
2. Choose **Create channel**.
3. Name it `home-test-medialive`.
4. Choose the MediaLive service role.
5. Attach the RTMP Push input.
6. Select a single-pipeline channel for a simpler/lower-cost test if acceptable.
7. Configure input specification to match the OBS source, such as HD 720p/1080p AVC.
8. Create an HLS output group.
9. Set the HLS destination to MediaPackage over HTTPS.
10. Configure a simple output:
    - 720p/30
    - AVC/H.264
    - AAC audio
    - Segment length appropriate for first test, such as 2–6 seconds
11. Create the channel.

AWS documents creating HLS output groups in MediaLive and sending HLS output to MediaPackage.[^medialive-create-hls-output][^medialive-hls-mediapackage]

**Expected result:** MediaLive channel is created but not yet running.

**Checkpoint:** Channel state is idle/stopped/created, and it has the RTMP input plus HLS output group.

---

### Step 9: Create a safe OBS test scene

1. Open OBS.
2. In **Scenes**, click **+** and create `AWS MediaLive Test`.
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

### Step 10: Configure OBS video/audio

Open **Settings → Video**:

| Setting | Starter value |
|---|---|
| Base canvas resolution | 1280×720 or your monitor resolution |
| Output scaled resolution | 1280×720 for easier testing |
| FPS | 30 |

Open **Settings → Output → Streaming**:

| Setting | Starter value |
|---|---|
| Encoder | Hardware encoder if stable; otherwise x264 |
| Rate control | CBR |
| Video bitrate | 2500–4500 Kbps for 720p/30 test |
| Keyframe interval | 2 seconds |
| Audio bitrate | 128–160 Kbps |
| Video codec | H.264/AVC |
| Audio codec | AAC |

**Expected result:** OBS is configured for a modest, stable test stream.

**Checkpoint:** Your total upload bitrate is comfortably below your available upload speed.

---

### Step 11: Configure OBS stream target

1. Open **Settings → Stream**.
2. Set **Service** to **Custom**.
3. Set **Server** to the MediaLive RTMP destination URL.
4. Set **Stream Key** to the stream name/path required by the MediaLive input destination.
5. Save.

**Important:** MediaLive RTMP Push destination formatting can be slightly different from simpler services. Use the exact destination URL and application/stream name shown in the MediaLive input details. If the AWS Console labels differ from this guide, use the current MediaLive RTMP Push input documentation and the values shown in your account.

**Expected result:** OBS is ready to push to MediaLive.

**Checkpoint:** OBS contains the correct MediaLive endpoint and stream path, and the stream key/path is not shared publicly.

---

### Step 12: Start the MediaLive channel

1. In MediaLive, open the channel.
2. Choose **Start**.
3. Wait for the channel to enter a running state.

AWS documentation says most AWS Cloud MediaLive channels start in 3 minutes or less, but up to 10 minutes can be normal.[^medialive-start-stop]

**Expected result:** The MediaLive channel is running and waiting for input.

**Checkpoint:** Channel state is running.

---

### Step 13: Start streaming in OBS

1. Click **Start Streaming** in OBS.
2. Watch the OBS status bar.
3. Look for:
   - Stable bitrate
   - Low or no dropped frames
   - No reconnect loop

**Expected result:** OBS pushes video to the MediaLive RTMP Push input.

**If it fails:**
- Check input security group.
- Check public IP address.
- Check RTMP destination URL and stream path/key.
- Check firewall/VPN.
- Lower bitrate.

**Checkpoint:** OBS indicates it is streaming and MediaLive input shows active/healthy where available.

---

### Step 14: Confirm MediaLive output reaches MediaPackage

1. Open MediaLive channel details.
2. Check channel alerts and output status.
3. Open MediaPackage v2.
4. Check the channel/origin endpoint activity.
5. Watch for 4xx/5xx errors, credential errors, or origin ingest problems.

**Expected result:** MediaPackage receives the HLS output from MediaLive.

**Checkpoint:** MediaPackage endpoint exists and has recent activity once MediaLive is running.

---

### Step 15: Watch the stream in a browser

Use the CloudFront URL that maps to the MediaPackage HLS endpoint.

For a simple test player, create `medialive-hls-test-player.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>AWS MediaLive HLS Test Player</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 2rem; }
    video { width: 100%; max-width: 960px; background: #000; }
  </style>
</head>
<body>
  <h1>AWS MediaLive HLS Test Player</h1>
  <p>Replace HLS_PLAYBACK_URL below with your CloudFront HLS URL.</p>
  <video id="video" controls autoplay muted playsinline></video>

  <script>
    const video = document.getElementById("video");
    const hlsUrl = "HLS_PLAYBACK_URL";

    if (video.canPlayType("application/vnd.apple.mpegurl")) {
      video.src = hlsUrl;
    } else if (Hls.isSupported()) {
      const hls = new Hls();
      hls.loadSource(hlsUrl);
      hls.attachMedia(video);
    } else {
      document.body.insertAdjacentHTML(
        "beforeend",
        "<p>This browser does not support HLS playback.</p>"
      );
    }
  </script>
</body>
</html>
```

Replace `HLS_PLAYBACK_URL` with your CloudFront HLS manifest URL.

**Expected result:** Your live stream plays in the browser.

**Checkpoint:** You can see and hear the live stream.

---

## Verification checkpoints

| Checkpoint | Success condition |
|---|---|
| AWS sign-in | You can access AWS Console, MediaLive, MediaPackage, CloudFront, and Billing. |
| Budget | AWS Budgets has a cost budget or alert. |
| MediaPackage | Channel/origin endpoint exists. |
| CloudFront | Distribution points to MediaPackage origin. |
| MediaLive input security group | Allows your public IP only. |
| MediaLive RTMP input | Input exists and has destination URL(s). |
| MediaLive channel | Channel has RTMP input and HLS output to MediaPackage. |
| OBS scene | OBS preview shows safe content. |
| OBS stream settings | Bitrate, FPS, resolution, and keyframe interval are reasonable. |
| MediaLive running | Channel starts successfully. |
| OBS connection | OBS says it is streaming without errors. |
| MediaPackage ingest | MediaPackage receives MediaLive output. |
| CloudFront playback | Browser player displays the stream. |
| Audio | Expected audio is present. |
| Latency | Delay is understood and acceptable. |
| Security | Input security group and stream details remain private. |
| Cost | You know how to stop/delete resources. |
| Cleanup | MediaLive channel is stopped and unused resources are deleted or intentionally retained. |

---

## Testing the stream

### Basic test

1. Start the MediaLive channel.
2. Start OBS streaming.
3. Open the HLS player page.
4. Confirm picture.
5. Confirm audio.
6. Move something visible in OBS to estimate latency.
7. Stop OBS.
8. Stop the MediaLive channel.
9. Refresh playback and confirm it eventually stops.

### Quality test

| Test | What to watch |
|---|---|
| 5-minute stream | Dropped frames, OBS reconnects, MediaLive alerts. |
| Lower bandwidth | Playback buffering or quality fallback. |
| Different browser | HLS compatibility and player errors. |
| Mobile device | iOS/Android playback behavior. |
| Private/incognito browser | Whether playback truly requires no login. |
| Shared network | Whether others on your home network cause upload instability. |
| Channel restart | How long MediaLive startup takes. |

### Upload bandwidth rule of thumb

Your upload speed should be significantly higher than your total stream bitrate. If OBS sends 4.5 Mbps video plus audio overhead, do not assume a 5 Mbps upload connection is enough. Leave headroom.

---

## Public playback setup

### What “public” means here

Public playback means viewers do not need to sign in. It does **not** mean:

- The OBS/MediaLive ingest details are public.
- AWS credentials are public.
- The MediaPackage origin should be directly exposed if CloudFront is the intended delivery layer.
- The stream is protected from being shared.
- Costs are capped automatically.

### Browser player choices

| Player approach | Use it when |
|---|---|
| Native Safari HLS | Quick test on Safari/iOS/macOS. |
| hls.js | Recommended generic browser test for HLS in Chrome/Edge/Firefox. |
| Video.js with HLS support | Useful if building a polished web player. |
| CloudFront URL | Recommended viewer-facing delivery URL. |
| MediaPackage origin URL | Useful for testing, but production should generally use CloudFront. |

### Hosting the player page

For a quick test, open the local HTML file on your PC. For sharing with others, host the page somewhere public, such as:

- Amazon S3 static website hosting plus CloudFront
- Your existing website
- A small static hosting service
- A local web server only for LAN testing

Do not put AWS credentials, MediaLive input details, or private secrets in the player page. Only the public HLS playback URL belongs in the browser page.

---

## Troubleshooting

| Symptom | Likely cause | How to confirm | Fix | Prevention |
|---|---|---|---|---|
| OBS cannot connect | Wrong RTMP URL/path or blocked IP | OBS log/status shows connection failure | Re-copy MediaLive input URL; update security group | Use the setup worksheet |
| MediaLive shows no input | Input security group blocks your IP | Check public IP and input SG CIDR | Add correct `/32` IP | Avoid dynamic IP surprises |
| Channel will not start | Configuration, IAM, output destination, or service quota issue | MediaLive alerts/errors | Fix channel validation errors; check role | Start with simple single output |
| MediaLive starts slowly | Normal startup behavior | Channel status remains starting | Wait up to documented normal range | Start early before event |
| MediaPackage receives nothing | MediaLive output destination/credentials wrong | MediaLive output alerts | Recheck MediaPackage destination | Create downstream first |
| CloudFront playback fails | Wrong origin/cache behavior/path | Browser/network errors | Verify MediaPackage endpoint and CloudFront config | Use documented CloudFront setup |
| HLS player does not load | Browser lacks native HLS or CORS/player issue | Browser console errors | Use hls.js; check URL/CORS | Test Safari + Chrome |
| Audio but no video | OBS/MediaLive encode mismatch | OBS preview or MediaLive alerts | Check video source/codec settings | Use H.264/AAC starter settings |
| Video but no audio | Audio source muted/wrong device | OBS mixer meters don’t move | Select correct mic/desktop source | Name audio sources |
| High latency | HLS segment length/player buffering | Compare OBS motion to playback | Tune segment length/LL-HLS later | Accept normal HLS for first test |
| Unexpected charges | Channel/input left idle/running | Check billing/usage | Stop/delete MediaLive resources | Use cleanup checklist |
| IAM permission denied | User/service role lacks permissions | Console or CLI error | Update role/policies | Use known MediaLive service role |
| AWS docs don’t match Console | UI changed | Menu labels differ | Use Console search; verify docs | Record Last verified date |
| Region mismatch | Resources created in different Regions | Missing resources or bad endpoints | Keep MediaLive/MediaPackage same Region | Write Region in worksheet |

---

## Cleanup, shutdown, and avoiding surprise charges

### Stop the stream

1. In OBS, click **Stop Streaming**.
2. Confirm OBS is no longer streaming.
3. In MediaLive, open the channel.
4. Choose **Stop**.
5. Wait for the channel to stop.

AWS MediaLive documentation says you can stop a running channel at any time, and that different charges apply depending on channel state.[^medialive-start-stop]

### Delete resources if done

| Resource | What to do |
|---|---|
| MediaLive channel | Stop first, then delete if not needed. |
| MediaLive RTMP Push input | Delete if not needed; push inputs can have idle costs.[^medialive-pricing-page] |
| MediaLive input security group | Delete if not needed. |
| MediaPackage channel/origin endpoint | Delete if test-only. |
| CloudFront distribution | Disable/delete if test-only; remember distribution deletion can take time. |
| S3 bucket/player hosting | Delete if created and no longer needed. |
| CloudWatch alarms/logs | Delete if test-only or intentionally retain. |
| AWS Budget | Keep it; useful for future AWS experiments. |

### Final cleanup checklist

- [ ] OBS stopped.
- [ ] MediaLive channel stopped.
- [ ] MediaLive channel deleted or intentionally retained.
- [ ] RTMP Push input deleted or intentionally retained.
- [ ] Input security group deleted or intentionally retained.
- [ ] MediaPackage v2 channel/origin endpoint deleted or intentionally retained.
- [ ] CloudFront distribution disabled/deleted or intentionally retained.
- [ ] HLS player page removed if no longer needed.
- [ ] Any S3 hosting bucket deleted or intentionally retained.
- [ ] Billing dashboard reviewed after usage updates.
- [ ] AWS Budget remains active.
- [ ] No stream endpoint, credential, or secret was shared publicly.

---

## Security notes

### Keep private

- AWS account credentials
- IAM access keys
- MediaLive service role details beyond what is needed
- MediaLive RTMP input destination/path
- OBS stream target settings
- MediaPackage origin configuration if CloudFront is intended as the public layer

### Public playback risk

If playback is public, anyone with the CloudFront HLS URL or embedded player page may be able to watch. If the URL is shared broadly, CloudFront and origin traffic costs can grow.

### Input security group

Restrict MediaLive RTMP Push input access to your public IP address where practical. If your ISP changes your public IP, update the input security group.

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
| Viewer authentication | Use CloudFront signed URLs/cookies, application auth, or CDN/origin authorization patterns. |
| Domain name | Put the player page and CloudFront distribution behind your own domain. |
| HTTPS/TLS | Serve player pages and playback over HTTPS. |
| Monitoring | Watch MediaLive alerts, MediaPackage origin health, CloudFront metrics, and OBS health. |
| Alarms | Create CloudWatch alarms for MediaLive errors, dropped input, spend, and distribution anomalies. |
| Logging | Enable/retain logs intentionally and understand log costs. |
| Cost alerts | Use AWS Budgets and review costs after every test. |
| Scaling | CloudFront handles viewer distribution, but origin, cache behavior, and cost matter. |
| Reliability | Consider standard channels/two pipelines only when the cost is justified. |
| Redundancy | Add backup encoder/network/inputs only when needed. |
| Stream health checks | Document preflight checks and channel startup time. |
| Runbook | Document start/stop steps, emergency stop, contact list, and cleanup. |
| Architecture review | Revisit IVS if simplicity/low latency/cost matter more than MediaLive workflow control. |

---

## Glossary

| Term | Meaning |
|---|---|
| OBS | Open Broadcaster Software Studio, local capture/encoding/streaming software. |
| Encoder | Software/hardware that compresses video/audio and sends it to a streaming service. |
| RTMP | Live ingest protocol used by OBS to push to MediaLive in this guide. |
| RTMP Push | Workflow where upstream system pushes stream into MediaLive. |
| HLS | HTTP Live Streaming, common browser/mobile playback format. |
| LL-HLS | Low-Latency HLS; reduced-latency version of HLS. |
| WebRTC | Low-latency real-time media protocol family. |
| Latency | Delay between capture and viewer playback. |
| Bitrate | Data rate used by audio/video. Higher can mean better quality but more upload and cost. |
| Frame rate | Frames per second, such as 30 or 60 FPS. |
| Keyframe interval | Time between full video frames. |
| Transcoding | Creating new encoded versions/renditions from the input stream. |
| Packaging | Preparing encoded media into playback formats such as HLS. |
| Origin | Source server/service for CDN delivery. |
| CDN | Content delivery network. |
| CloudFront | AWS CDN often used for HTTP video delivery. |
| MediaLive | AWS real-time video service for broadcast and streaming delivery. |
| MediaPackage v2 | AWS media packaging/origin service used for HLS/LL-HLS endpoints. |
| IVS | Amazon Interactive Video Service; simpler managed live-video option. |
| MediaConnect | AWS service for reliable live video transport and routing. |
| IAM | AWS Identity and Access Management. |
| Security group | In this guide, MediaLive input security group controls allowed source IPs. |
| Public playback URL | URL viewers can use to watch without login. |
| Ingest URL | URL used by OBS/encoder to send video. |
| Stream key/path | OBS stream target path/name; treat as private. |

---

## Index

| Keyword | Where to look |
|---|---|
| AWS Budgets | Cost warning; Step 2; Appendix B |
| AWS CLI | Appendix C; Appendix D |
| Browser playback | Public playback setup; Step 15 |
| Cleanup | Cleanup section; Appendix J |
| CloudFront | Step 5; Public playback; Appendix A |
| Cost | Cost warning; Cleanup; Appendix G |
| EC2 | Brief comparison; Appendix A |
| HLS | MediaPackage; Public playback; Glossary |
| IAM | Step 3; Security notes |
| Input security group | Step 6; Troubleshooting |
| Latency | Known assumptions; Testing; Production hardening |
| MediaConnect | Brief comparison; Appendix A |
| MediaLive | Recommended path; Step 7–14 |
| MediaPackage v2 | Step 4; Step 14; Appendix A |
| OBS | Prerequisites; Step 9–13 |
| Playback URL | Step 15; Public playback |
| Public playback | Public playback definition; Public playback setup |
| Region mismatch | Step 1; Troubleshooting |
| RTMP Push | Step 7; OBS Stream target |
| Security | Security notes; Production hardening |
| Stream key/path | Step 11; Security notes |
| Troubleshooting | Troubleshooting section; Appendix E |

---

# Appendixes

<details>
<summary><strong>Appendix A: Detailed architecture comparison</strong></summary>

## 1. OBS to MediaLive + MediaPackage v2 + CloudFront — recommended for this version

- **Overview:** Broadcast-style AWS live pipeline.
- **How it works:** OBS pushes RTMP to MediaLive; MediaLive outputs HLS to MediaPackage; CloudFront delivers to viewers.
- **Services involved:** MediaLive, MediaPackage v2, CloudFront, IAM, CloudWatch, AWS Budgets.
- **Setup complexity:** High.
- **Operational complexity:** Medium/high.
- **Cost behavior:** MediaLive input/output and idle resource charges; MediaPackage and CloudFront add usage costs.[^medialive-pricing-doc][^medialive-pricing-page]
- **Latency:** Standard HLS by default; LL-HLS possible with additional tuning.
- **Pros:** Flexible, broadcast-oriented, teaches AWS media pipeline.
- **Cons:** More expensive/complex than IVS.
- **Best use:** Learning or operating MediaLive-centered workflows.
- **Recommendation:** Best fit only because this version is MediaLive-focused.

## 2. OBS to Amazon IVS — simpler alternative

- **Overview:** Managed AWS live-streaming service with ingest and playback.
- **Setup complexity:** Low.
- **Cost behavior:** IVS input/output pricing.
- **Latency:** AWS describes IVS Low-Latency Streaming as under five seconds using RTMPS delivery.[^ivs-what-is]
- **Pros:** Much simpler than MediaLive for OBS-to-browser streaming.
- **Cons:** Less broadcast-pipeline control.
- **Recommendation:** Usually better if the goal is simplest practical AWS streaming.

## 3. OBS to EC2/Lightsail with NGINX RTMP — self-managed alternative

- **Overview:** Build your own RTMP/HLS server.
- **Pros:** Educational and flexible.
- **Cons:** You manage security, patching, scaling, HLS, and data transfer.
- **Recommendation:** Not first path for AWS managed media.

## 4. OBS to MediaConnect — transport/routing alternative

- **Overview:** Reliable live video transport and routing.
- **Pros:** Useful for broadcast contribution.
- **Cons:** Does not itself provide browser playback.
- **Recommendation:** Not for this beginner public-browser goal.

## 5. Non-AWS relay

- **Overview:** Twitch/YouTube/Vimeo/etc.
- **Pros:** Very simple.
- **Cons:** Not AWS MediaLive and not AWS-hosted.
- **Recommendation:** Mentioned only for context.

</details>

<details>
<summary><strong>Appendix B: AWS services used or referenced</strong></summary>

| Service | Required? | Why mentioned | Cost notes |
|---|---:|---|---|
| MediaLive | Yes | Main live encoder/transcoder. | Input/output and idle-resource charges may apply. |
| MediaPackage v2 | Yes | HLS origin/packager. | Packaging/origin/request/egress charges. |
| CloudFront | Yes for public CDN delivery | Public distribution of HLS stream. | Data transfer and request charges. |
| IAM | Yes | Service roles and user permissions. | No direct cost for normal IAM use. |
| AWS Budgets | Recommended | Alerts on spend. | Check current AWS cost management terms. |
| CloudWatch | Optional/recommended | Metrics, logs, alarms. | Metrics/logs/alarms may cost. |
| Amazon IVS | Alternative | Simpler AWS live-streaming path. | Input/output pricing. |
| EC2 | Alternative | Self-managed RTMP server. | Instance and data transfer. |
| Lightsail | Alternative | Simplified VPS for RTMP server. | Monthly bundle plus transfer overage. |
| MediaConnect | Alternative | Broadcast video transport. | Flow/output/data pricing. |

</details>

<details>
<summary><strong>Appendix C: AWS CLI command reference</strong></summary>

> Console instructions are primary. MediaLive and MediaPackage CLI setup can be verbose and easier to misconfigure. Use CLI only if you are comfortable.

## `aws configure`

```powershell
aws configure
```

Purpose: Configure AWS CLI credentials and default Region.

## `aws medialive create-input-security-group`

Purpose: Create an input security group for MediaLive.

```powershell
aws medialive create-input-security-group ^
  --whitelist-rules Cidr=203.0.113.25/32 ^
  --region us-west-2
```

## `aws medialive create-input`

Purpose: Create a MediaLive input.

```powershell
aws medialive create-input ^
  --name obs-rtmp-push-input ^
  --type RTMP_PUSH ^
  --input-security-groups INPUT_SECURITY_GROUP_ID ^
  --destinations StreamName=live/test ^
  --region us-west-2
```

The exact CLI shape can vary by desired input type and destination structure. Verify against current AWS CLI docs before using.

## `aws medialive create-channel`

Purpose: Create a MediaLive channel.

This command requires a large JSON payload. For novice users, use the Console first. Export or document the channel later if you need automation.

## `aws medialive start-channel`

```powershell
aws medialive start-channel --channel-id CHANNEL_ID --region us-west-2
```

## `aws medialive stop-channel`

```powershell
aws medialive stop-channel --channel-id CHANNEL_ID --region us-west-2
```

## `aws medialive delete-channel`

```powershell
aws medialive delete-channel --channel-id CHANNEL_ID --region us-west-2
```

## `aws cloudfront create-distribution`

Purpose: Create a CloudFront distribution. Usually easier through Console for the first setup because cache behaviors/origins are detailed.

## `aws mediapackagev2`

Purpose: Manage MediaPackage v2 resources. Command details can change; verify current AWS CLI v2 documentation for `mediapackagev2` before automating.

</details>

<details>
<summary><strong>Appendix D: AWS CLI task sequences</strong></summary>

## Task: Start/stop an existing MediaLive channel

```powershell
aws medialive start-channel --channel-id CHANNEL_ID --region us-west-2
```

Wait for the channel to run, then start OBS.

```powershell
aws medialive stop-channel --channel-id CHANNEL_ID --region us-west-2
```

Use this after OBS is stopped.

## Task: Delete a test MediaLive channel

```powershell
aws medialive stop-channel --channel-id CHANNEL_ID --region us-west-2
aws medialive delete-channel --channel-id CHANNEL_ID --region us-west-2
```

Delete inputs and input security groups afterward if unused.

</details>

<details>
<summary><strong>Appendix E: Troubleshooting quick reference</strong></summary>

- **OBS reconnect loop:** Check RTMP URL/path, input security group, firewall/VPN.
- **MediaLive input missing:** Confirm public IP CIDR and input destination.
- **Channel validation error:** Simplify output group and verify MediaPackage destination.
- **MediaPackage no output:** Check MediaLive output credentials and destination URL.
- **CloudFront 403/404:** Check origin domain, path, cache behavior, and MediaPackage endpoint.
- **High latency:** Standard HLS has latency; tune LL-HLS later.
- **Billing surprise:** Stop/delete MediaLive channels and RTMP Push inputs; check MediaPackage/CloudFront too.

</details>

<details>
<summary><strong>Appendix F: Glossary expansion</strong></summary>

See [Glossary](#glossary). Add new terms to this appendix as the guide evolves.

</details>

<details>
<summary><strong>Appendix G: Source links and last-verified notes</strong></summary>

**Last verified:** June 28, 2026.

Official sources used:

[^medialive-what-is]: What is AWS Elemental MediaLive: https://docs.aws.amazon.com/medialive/latest/ug/what-is.html  
[^medialive-rtmp-push]: Setting up an RTMP Push input in MediaLive: https://docs.aws.amazon.com/medialive/latest/ug/input-create-rtmp-push.html  
[^medialive-hls-mediapackage]: HLS output group to MediaPackage: https://docs.aws.amazon.com/medialive/latest/ug/origin-server-hls-emp.html  
[^medialive-create-hls-output]: MediaLive getting-started HLS output group: https://docs.aws.amazon.com/medialive/latest/ug/getting-started-step5.html  
[^medialive-start-stop]: Starting, stopping, and pausing a MediaLive channel: https://docs.aws.amazon.com/medialive/latest/ug/starting-stopping-deleting-a-channel.html  
[^medialive-pricing-doc]: Pricing in MediaLive docs: https://docs.aws.amazon.com/medialive/latest/ug/pricing.html  
[^medialive-pricing-page]: AWS Elemental MediaLive pricing: https://aws.amazon.com/medialive/pricing/  
[^mediapackage-hls-llhls]: MediaPackage v2 HLS and LL-HLS overview: https://docs.aws.amazon.com/mediapackage/latest/userguide/hls-overview.html  
[^mediapackage-pricing]: MediaPackage pricing: https://aws.amazon.com/mediapackage/pricing/  
[^cloudfront-live-mediapackage]: CloudFront live streaming with AWS Media Services: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/live-streaming.html  
[^cloudfront-pricing]: CloudFront pricing: https://aws.amazon.com/cloudfront/pricing/  
[^mediaconnect-what-is]: What is MediaConnect: https://docs.aws.amazon.com/mediaconnect/latest/ug/what-is.html  
[^ivs-what-is]: Amazon IVS Low-Latency Streaming overview: https://docs.aws.amazon.com/ivs/latest/LowLatencyUserGuide/what-is.html  
[^obs-download]: OBS Studio download page: https://obsproject.com/download  
[^aws-cli-getting-started]: AWS CLI getting started: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html  
[^iam-best-practices]: IAM security best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html  
[^iam-root]: Root user best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html  
[^aws-budgets]: Managing costs with AWS Budgets: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html  
[^aws-regional-services]: AWS Regional Services list: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/  

Known likely-to-change areas:

- AWS Console labels and navigation.
- MediaPackage v2 Console flow.
- MediaLive pricing and idle-resource charges.
- CloudFront distribution settings.
- Service quotas.
- Regional availability.
- Browser/player behavior.

</details>

<details>
<summary><strong>Appendix H: My setup values worksheet</strong></summary>

| Field | Value |
|---|---|
| AWS Region |  |
| Recommended architecture selected | MediaLive + MediaPackage v2 + CloudFront |
| MediaLive channel name |  |
| MediaLive channel ID |  |
| MediaLive input name |  |
| MediaLive input destination URL |  |
| OBS stream path/key | **Do not paste into shared docs** |
| MediaLive input security group ID |  |
| Allowed public IP CIDR |  |
| MediaPackage channel group |  |
| MediaPackage channel |  |
| MediaPackage origin endpoint |  |
| CloudFront distribution ID |  |
| CloudFront domain |  |
| HLS playback URL |  |
| Test player file | `medialive-hls-test-player.html` |
| OBS scene name |  |
| Resolution |  |
| Frame rate |  |
| Video bitrate |  |
| Audio bitrate |  |
| Keyframe interval |  |
| Expected latency |  |
| Test start time |  |
| Test stop time |  |
| Resources to stop | OBS stream, MediaLive channel |
| Resources to delete | MediaLive channel/input, MediaPackage, CloudFront if done |

</details>

<details>
<summary><strong>Appendix I: Decision log</strong></summary>

| Decision | Reason |
|---|---|
| Use MediaLive as primary path | Requested MediaLive + OBS version. |
| Include MediaPackage v2 | MediaLive needs a downstream origin/packager for public HLS playback. |
| Include CloudFront | Recommended public CDN delivery layer for MediaPackage playback. |
| Use RTMP Push | OBS can push RTMP, and MediaLive documents RTMP Push inputs. |
| Use HLS for first playback test | Widely supported and documented with MediaPackage/CloudFront. |
| Defer LL-HLS | Lower latency adds configuration complexity. |
| Do not use IVS as primary | This version intentionally focuses on MediaLive. |
| Use Console first | MediaLive/MediaPackage setup is detailed and easier to understand visually. |

</details>

<details>
<summary><strong>Appendix J: Final cleanup checklist</strong></summary>

- [ ] OBS stopped.
- [ ] MediaLive channel stopped.
- [ ] MediaLive channel deleted or intentionally retained.
- [ ] RTMP Push input deleted or intentionally retained.
- [ ] Input security group deleted or intentionally retained.
- [ ] MediaPackage v2 resources deleted or intentionally retained.
- [ ] CloudFront distribution disabled/deleted or intentionally retained.
- [ ] HLS player page removed if no longer needed.
- [ ] Any S3 hosting bucket deleted or intentionally retained.
- [ ] CloudWatch alarms/logs deleted or intentionally retained.
- [ ] Billing dashboard reviewed after usage updates.
- [ ] AWS Budget remains active.

</details>

<details>
<summary><strong>Appendix K: Screenshot and visual aid suggestions</strong></summary>

| Visual | Where it belongs | Why it helps |
|---|---|---|
| Architecture diagram | Brief summary | Shows PC → MediaLive → MediaPackage → CloudFront → browser. |
| AWS Region selector | Step 1 | Prevents Region mismatch. |
| MediaPackage endpoint page | Step 4 | Shows where origin/playback details come from. |
| CloudFront origin/cache behavior | Step 5 | Helps with delivery configuration. |
| MediaLive input security group | Step 6 | Shows source IP allowlist. |
| MediaLive RTMP Push input | Step 7 | Shows OBS destination information. |
| MediaLive channel output group | Step 8 | Shows HLS to MediaPackage setup. |
| OBS Stream settings | Step 11 | Shows where Server and Stream Key/path go. |
| HLS player page | Step 15 | Confirms what success looks like. |
| Cleanup checklist screenshot | Cleanup section | Reinforces cost control. |

</details>

<details>
<summary><strong>Appendix L: Expandable appendix guidance</strong></summary>

This guide uses expandable `<details>` sections for appendixes. Essential setup instructions remain in the main guide. Appendixes provide deeper reference material, extended comparisons, and checklists.

</details>
