# OBS to AWS Elemental MediaLive Cheat Sheet

**Version:** v1.1.0  
**Full guide:** [`aws-live-streaming-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-guide-obs-medialive-v1.1.0.md)  
**Quick-start guide:** [`aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md)

## Path

```text
OBS → MediaLive RTMP Push input → MediaLive channel → MediaPackage v2 → CloudFront → browser
```

## Use this when

You specifically want the more advanced AWS MediaLive broadcast-style workflow. If you only need a simple OBS-to-browser AWS stream, use the IVS quick start instead.

## Minimum prerequisites

- Windows 11 PC
- OBS Studio
- AWS account
- Billing access
- Permission to create MediaLive, MediaPackage v2, CloudFront, IAM roles, and budgets
- MediaLive service role or permission to create/use one
- Safe test scene
- Stable upload connection

## Fast setup

1. Sign in to AWS.
2. Choose one Region.
3. Create an AWS Budget.
4. Confirm IAM and MediaLive service-role access.
5. Create MediaPackage v2 channel/origin endpoint.
6. Create CloudFront distribution pointing to MediaPackage.
7. Create MediaLive input security group for your public IP `/32`.
8. Create MediaLive RTMP Push input.
9. Create MediaLive channel:
   - RTMP Push input
   - HLS output group
   - Output to MediaPackage
   - Single pipeline for simplest test
10. Install/open OBS.
11. Create a safe test scene.
12. Configure OBS:
    - 1280×720
    - 30 FPS
    - 2500–4500 Kbps
    - H.264 video
    - AAC audio
    - 2-second keyframe interval
13. Set OBS stream:
    - Service: Custom
    - Server: MediaLive RTMP destination
    - Stream Key/path: MediaLive stream path
14. Start OBS first.
15. Start MediaLive channel.
16. Confirm MediaLive receives input.
17. Confirm MediaPackage receives output.
18. Open CloudFront HLS URL in browser/player.
19. Stop OBS.
20. Stop MediaLive channel.
21. Delete test-only resources.

## Do not share

- AWS credentials
- MediaLive input details
- OBS stream path/key
- Service-role credentials or IAM access keys

## Cleanup

- Stop OBS.
- Stop MediaLive channel.
- Delete MediaLive channel/input/security group if done.
- Delete MediaPackage resources if done.
- Disable/delete CloudFront distribution if done.
- Check billing later.
- Keep AWS Budget active.

## Common fixes

| Problem | Fix |
|---|---|
| OBS cannot connect | Check RTMP URL, stream path, allowed IP, firewall/VPN. |
| MediaLive no input | Start OBS first and confirm input security group. |
| Channel will not start | Check IAM role and validation errors. |
| No playback | Check MediaPackage endpoint and CloudFront path. |
| High latency | Standard HLS has latency; tune later. |
| Permission error | See full guide Appendix M. |
