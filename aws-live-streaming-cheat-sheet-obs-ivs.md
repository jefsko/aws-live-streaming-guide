# OBS to Amazon IVS Cheat Sheet

**Version:** v1.1.0  
**Full guide:** [`aws-live-streaming-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-guide-obs-ivs-v1.1.0.md)  
**Quick-start guide:** [`aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md)

## Path

```text
OBS → Amazon IVS → IVS playback URL → browser
```

## Use this when

You want the simplest AWS-native live-streaming path from OBS to a public browser-viewable stream.

## Minimum prerequisites

- Windows 11 PC
- OBS Studio
- AWS account
- Billing access
- Permission to create Amazon IVS resources
- Safe test scene
- Stable upload connection

## Fast setup

1. Sign in to AWS.
2. Choose one Region.
3. Create an AWS Budget.
4. Open Amazon IVS.
5. Create an IVS channel.
6. Copy:
   - Ingest endpoint
   - Stream key
   - Playback URL
7. Install/open OBS.
8. Create a safe test scene.
9. Set OBS video:
   - 1280×720
   - 30 FPS
   - 2500–4500 Kbps
   - 2-second keyframe interval
10. Set OBS stream:
    - Service: Custom
    - Server: IVS ingest endpoint
    - Stream Key: IVS stream key
11. Start streaming in OBS.
12. Confirm IVS receives the stream.
13. Open playback URL in the IVS player.
14. Stop OBS.
15. Confirm IVS is inactive.
16. Delete the IVS channel if done.

## Do not share

- AWS credentials
- IVS stream key
- Screenshots showing secret values

## Cleanup

- Stop OBS.
- Confirm IVS stream inactive.
- Delete IVS channel if not needed.
- Check billing later.
- Keep AWS Budget active.

## Common fixes

| Problem | Fix |
|---|---|
| OBS cannot connect | Re-copy endpoint/key. |
| No IVS activity | Check Region and channel. |
| Browser will not play | Use IVS player sample from full guide. |
| Poor quality | Lower bitrate/resolution. |
| Permission error | See full guide Appendix M. |
