# AWS Live Streaming Quick Start Guide: OBS to AWS Elemental MediaLive

**Version:** v1.1.0  
**Based on full guide:** [`aws-live-streaming-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-guide-obs-medialive-v1.1.0.md)  
**Recommended path:** OBS Studio on Windows 11 → AWS Elemental MediaLive → MediaPackage v2 → CloudFront → browser playback  
**Best for:** Learning or using a more advanced AWS broadcast-style live streaming workflow.

This quick-start guide is the short version. It keeps the MediaLive path as simple as possible, but MediaLive is still more advanced than Amazon IVS. If your goal is simply to get OBS live in a browser through AWS, start with the IVS quick-start guide instead.

---

## Before you start

You need a Windows 11 PC, OBS Studio, an AWS account, billing access, and permission to create MediaLive, MediaPackage v2, CloudFront, IAM, and optional CloudWatch resources.

For AWS access, use IAM Identity Center or an IAM user/group with appropriate permissions. For MediaLive, you may also need a MediaLive service role and permission to pass that role. For exact IAM setup, see the full guide:

- Full guide: **Step 0: Set up AWS access, IAM, roles, and permissions**
- Full guide: **Step 3: Create or identify the MediaLive service role**
- Full guide: **Appendix M: Detailed IAM setup**

Use non-sensitive test content. MediaLive costs can add up faster than the IVS path, so keep the first test short.

---

## What this builds

```text
OBS Studio on Windows 11
   ↓ RTMP Push
AWS Elemental MediaLive input and channel
   ↓ HLS output
AWS Elemental MediaPackage v2
   ↓
Amazon CloudFront
   ↓
Browser playback with HLS
```

Viewers can open the CloudFront playback URL or player page without logging in. The OBS/MediaLive input details, AWS credentials, service roles, and origin configuration should stay private.

---

## Cost warning

This path can create charges from several services:

- MediaLive channel running or idle resources
- MediaLive RTMP Push input
- MediaPackage v2
- CloudFront data transfer and requests
- CloudWatch logs, metrics, or alarms if enabled

Stopping OBS is **not enough**. You must also stop the MediaLive channel and delete unused resources if this was only a test.

See the full guide’s billing and cleanup sections for details.

---

## Quick-start steps

### 1. Sign in to AWS and choose a Region

1. Open the AWS Management Console.
2. Choose a Region, such as `us-west-2`.
3. Use the same Region for MediaLive and MediaPackage where applicable.

**Checkpoint:** The AWS Console Region selector shows your intended Region.

---

### 2. Set up a quick billing safety net

1. Open **Billing and Cost Management**.
2. Open **Budgets**.
3. Create a small cost budget alert.
4. Add your email address.

**Checkpoint:** You have at least one budget alert configured.

---

### 3. Confirm MediaLive IAM/service-role readiness

1. Open **AWS Elemental MediaLive**.
2. During channel creation, watch for a prompt to create or select a MediaLive service role.
3. Use the Console-created role if this is a personal lab account and the Console offers it.
4. In stricter environments, ask your AWS administrator for the approved MediaLive service role.

For full details, see:

- Full guide: **Step 3: Create or identify the MediaLive service role**
- Full guide: **Appendix M: Detailed IAM setup**

**Checkpoint:** You know which MediaLive role the channel will use.

---

### 4. Create the MediaPackage v2 destination

Create the downstream origin before the MediaLive channel.

1. Open **AWS Elemental MediaPackage**.
2. Use the current MediaPackage v2 workflow.
3. Create a channel group if required.
4. Create a channel.
5. Create an HLS origin endpoint.
6. Record the MediaPackage endpoint/origin details needed by MediaLive.

**Checkpoint:** MediaPackage has an HLS endpoint ready for MediaLive output.

---

### 5. Create a CloudFront distribution

1. Open **CloudFront**.
2. Create a distribution.
3. Set the MediaPackage endpoint as the origin.
4. Use HTTPS.
5. Record the CloudFront distribution domain name.
6. Wait for CloudFront to deploy.

**Checkpoint:** You have a CloudFront domain pointing to MediaPackage.

---

### 6. Create a MediaLive input security group

1. Find your public IPv4 address.
2. Open **AWS Elemental MediaLive**.
3. Go to **Input security groups**.
4. Create an input security group.
5. Add your public IP as a `/32` CIDR.
   - Example: `203.0.113.25/32`
6. Avoid `0.0.0.0/0` unless this is a very temporary lab test.

**Checkpoint:** The input security group allows only your current public IP.

---

### 7. Create a MediaLive RTMP Push input

1. In MediaLive, open **Inputs**.
2. Choose **Create input**.
3. Select **RTMP Push**.
4. Name it `obs-rtmp-push-input`.
5. Attach the input security group.
6. Create the input.
7. Record the destination URL and stream name/path values.

**Checkpoint:** You have MediaLive RTMP Push destination details for OBS.

---

### 8. Create a MediaLive channel

1. In MediaLive, open **Channels**.
2. Choose **Create channel**.
3. Name it `home-test-medialive`.
4. Select the MediaLive service role.
5. Attach the RTMP Push input.
6. Use a single-pipeline setup for the simplest test if acceptable.
7. Add an HLS output group.
8. Send HLS output to MediaPackage v2.
9. Use a simple starter output:
   - 720p
   - 30 FPS
   - H.264 video
   - AAC audio
   - 2–6 second segment length
10. Create the channel, but do not start it yet.

**Checkpoint:** The MediaLive channel exists and is stopped/idle.

---

### 9. Install and configure OBS

1. Download and install OBS Studio.
2. Create a scene named `AWS MediaLive Test`.
3. Add a safe source, such as a Color Source.
4. Open **Settings → Video**:
   - Output resolution: `1280×720`
   - FPS: `30`
5. Open **Settings → Output → Streaming**:
   - Rate control: `CBR`
   - Video bitrate: `2500–4500 Kbps`
   - Audio bitrate: `128–160 Kbps`
   - Keyframe interval: `2 seconds`
   - Video codec: `H.264`
   - Audio codec: `AAC`

**Checkpoint:** OBS preview shows safe content and stream settings are modest.

---

### 10. Configure OBS to push to MediaLive

1. Open **Settings → Stream**.
2. Set **Service** to **Custom**.
3. Set **Server** to the MediaLive RTMP destination URL.
4. Set **Stream Key** to the stream name/path required by the MediaLive input.
5. Save.

**Checkpoint:** OBS has the correct MediaLive RTMP destination and stream path.

---

### 11. Start OBS first

For MediaLive RTMP Push, start OBS pushing before starting the MediaLive channel.

1. In OBS, click **Start Streaming**.
2. Watch for connection errors.
3. If OBS cannot connect, recheck the MediaLive input URL, stream path, and input security group.

**Checkpoint:** OBS is attempting to push to the MediaLive input.

---

### 12. Start the MediaLive channel

1. Open the MediaLive channel.
2. Choose **Start**.
3. Wait for the channel to reach a running state.
4. Be patient. MediaLive startup can take several minutes.

**Checkpoint:** MediaLive channel is running and receiving input.

---

### 13. Confirm output reaches MediaPackage and CloudFront

1. Check MediaLive alerts.
2. Check MediaPackage endpoint activity.
3. Open the CloudFront HLS URL or player page.

**Checkpoint:** The stream is available through the CloudFront playback URL.

---

### 14. Watch the stream in a browser

Use the HLS player from the full guide:

- Full guide: **Step 15: Watch the stream in a browser**
- Full guide: **Public playback setup**

Use Safari for the simplest native HLS test, or use the hls.js player example for Chrome, Edge, or Firefox.

**Checkpoint:** You can see and hear the stream in a browser.

---

## Stop here if your goal is only a quick test

If your goal was only to confirm OBS → MediaLive → MediaPackage → CloudFront playback:

1. Stop OBS.
2. Stop the MediaLive channel.
3. Delete the MediaLive channel if you do not need it.
4. Delete the RTMP Push input if you do not need it.
5. Delete the input security group if you do not need it.
6. Delete MediaPackage resources if this was only a test.
7. Disable/delete CloudFront if this was only a test.
8. Check billing/usage later.
9. Keep the AWS Budget active.

You are done.

---

## If something fails

| Problem | Most likely fix |
|---|---|
| OBS cannot connect | Check RTMP URL, stream path, input security group, firewall, and VPN. |
| MediaLive shows no input | Confirm OBS started first and your public IP is allowed. |
| Channel will not start | Check MediaLive role, validation errors, and output destination. |
| MediaPackage receives nothing | Recheck MediaLive HLS output destination. |
| CloudFront playback fails | Check origin, cache behavior, deployment status, and HLS path. |
| High latency | Standard HLS has latency; LL-HLS tuning belongs in the full guide. |
| Unexpected costs | Stop MediaLive, stop OBS, delete unused inputs/channels/origins/distributions. |
| IAM/permission error | See the full guide Step 0, Step 3, and Appendix M. |

---

## Where to read more

Use the full guide when you need:

- Exact IAM and MediaLive service-role setup
- More detailed AWS Console explanations
- MediaPackage and CloudFront details
- CLI examples
- Troubleshooting tables
- Security notes
- Cleanup details
- Production hardening

Full guide: [`aws-live-streaming-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-guide-obs-medialive-v1.1.0.md)
