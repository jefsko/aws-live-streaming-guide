# AWS Live Streaming Guides: OBS to IVS and MediaLive

Step-by-step guides for live streaming from OBS to AWS using Amazon IVS or AWS Elemental MediaLive.

This repository contains full guides, quick-start guides, and one-page cheat sheets for sending a live audio/video stream from **OBS Studio** to AWS and making the stream viewable in a browser.

The guides are written for technical users who are new to streaming engineering. They emphasize clear setup steps, AWS Console walkthroughs, billing awareness, verification checkpoints, troubleshooting, security notes, and cleanup steps.

## Which file should I open first?

| Goal | Start here |
|---|---|
| I want the simplest AWS-native OBS streaming path. | [`aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md) |
| I want a quick reminder after I already understand the IVS path. | [`aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md`](aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md) |
| I need the full IVS explanation, IAM setup, troubleshooting, and appendixes. | [`aws-live-streaming-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-guide-obs-ivs-v1.1.0.md) |
| I specifically need AWS Elemental MediaLive. | [`aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md) |
| I want a quick reminder after I already understand the MediaLive path. | [`aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md`](aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md) |
| I need the full MediaLive explanation, IAM/service-role setup, troubleshooting, and appendixes. | [`aws-live-streaming-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-guide-obs-medialive-v1.1.0.md) |

## Guides

| File | Type | Recommended path | Best for |
|---|---|---|---|
| [`aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md) | Quick start | OBS Studio → Amazon IVS → browser playback | First-time setup using the simpler AWS-native route |
| [`aws-live-streaming-guide-obs-ivs-v1.1.0.md`](aws-live-streaming-guide-obs-ivs-v1.1.0.md) | Full guide | OBS Studio → Amazon IVS → browser playback | Detailed IVS explanation, IAM setup, troubleshooting, security, cleanup |
| [`aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md`](aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md) | Cheat sheet | OBS Studio → Amazon IVS → browser playback | Fast repeat reference |
| [`aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md) | Quick start | OBS Studio → MediaLive → MediaPackage v2 → CloudFront → browser playback | First-time setup using the advanced MediaLive route |
| [`aws-live-streaming-guide-obs-medialive-v1.1.0.md`](aws-live-streaming-guide-obs-medialive-v1.1.0.md) | Full guide | OBS Studio → MediaLive → MediaPackage v2 → CloudFront → browser playback | Detailed MediaLive explanation, IAM/service-role setup, troubleshooting, security, cleanup |
| [`aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md`](aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md) | Cheat sheet | OBS Studio → MediaLive → MediaPackage v2 → CloudFront → browser playback | Fast repeat reference |

## Which guide should I use?

Start with the **Amazon IVS quick-start guide** if your main goal is to get an OBS stream publicly viewable in a browser with the least AWS complexity.

Use the **AWS Elemental MediaLive quick-start guide** if you specifically want to learn or use a more advanced AWS media pipeline involving MediaLive, MediaPackage v2, and CloudFront.

Use the **full guides** when you need more background, IAM instructions, AWS Console explanations, troubleshooting, security guidance, cleanup details, CLI notes, or appendixes.

Use the **cheat sheets** only after you understand the workflow and want a compact repeat checklist.

## Target audience

These guides are intended for:

- Windows 11 users running OBS Studio
- Technical beginners to live streaming architecture
- Developers, IT users, or hobbyists learning AWS media services
- Users who want explicit AWS Console steps with CLI notes as a secondary reference
- Users who care about billing, cleanup, troubleshooting, and avoiding surprise AWS charges

## Topics covered

The full guides cover:

- OBS Studio setup on Windows 11
- AWS account and billing prerequisites
- AWS IAM/access setup guidance
- Public browser playback concepts
- Amazon IVS live streaming
- AWS Elemental MediaLive live streaming
- MediaPackage v2 and CloudFront delivery
- HLS playback
- AWS CLI references where useful
- Verification checkpoints
- Troubleshooting
- Security considerations
- Cleanup and cost-control steps
- Glossaries, indexes, and appendixes

The quick-start guides condense the full workflows into shorter, sequential setup paths.

The cheat sheets provide one-page repeat-use reminders.

## Repository contents

```text
README.md
CHANGELOG.md
aws-live-streaming-guide-obs-ivs-v1.1.0.md
aws-live-streaming-guide-obs-medialive-v1.1.0.md
aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md
aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md
aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md
aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md
```

## Version

Current version: **v1.1.0**

The quick-start guides and cheat sheets are based on the v1.1.0 full guides.

## Recommended commit message

```text
Add quick-start guides and cheat sheets
```

## Notes

AWS service names, Console workflows, pricing, limits, and supported features can change over time. The full guides include last-verified dates and source links for important AWS and OBS references.

Always verify current AWS pricing and service behavior before running long live streams or leaving AWS resources active.
