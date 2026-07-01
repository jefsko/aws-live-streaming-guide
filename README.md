# AWS Live Streaming Guides: OBS to IVS and MediaLive

Step-by-step guides for live streaming from OBS to AWS using Amazon IVS or AWS Elemental MediaLive.

This repository contains two step-by-step AWS live streaming guides for sending a live audio/video stream from **OBS Studio** to AWS and making the stream viewable in a browser.

The guides are written for technical users who are new to streaming engineering. They emphasize clear setup steps, AWS Console walkthroughs, billing awareness, verification checkpoints, troubleshooting, security notes, and cleanup steps.

## Guides

| Guide | Recommended path | Best for |
|---|---|---|
| [`aws-live-streaming-guide-obs-ivs.md`](aws-live-streaming-guide-obs-ivs.md) | OBS Studio → Amazon IVS → browser playback | Simpler, lower-complexity AWS-native live streaming |
| [`aws-live-streaming-guide-obs-medialive.md`](aws-live-streaming-guide-obs-medialive.md) | OBS Studio → AWS Elemental MediaLive → MediaPackage v2 → CloudFront → browser playback | More advanced broadcast-style AWS media workflows |

## Which guide should I use?

Start with the **Amazon IVS guide** if your main goal is to get an OBS stream publicly viewable in a browser with the least AWS complexity.

Use the **AWS Elemental MediaLive guide** if you specifically want to learn or use a more advanced AWS media pipeline involving MediaLive, MediaPackage, and CloudFront.

## Target audience

These guides are intended for:

- Windows 11 users running OBS Studio
- Technical beginners to live streaming architecture
- Developers, IT users, or hobbyists learning AWS media services
- Users who want explicit AWS Console steps with CLI notes as a secondary reference
- Users who care about billing, cleanup, troubleshooting, and avoiding surprise AWS charges

## Topics covered

The guides cover:

- OBS Studio setup on Windows 11
- AWS account and billing prerequisites
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

## Repository contents

```text
README.md
CHANGELOG.md
aws-live-streaming-guide-obs-ivs.md
aws-live-streaming-guide-obs-medialive.md
```

## Version

Current version: **v1.1.0**

This repository currently contains the two companion guide variants:

1. OBS to Amazon IVS
2. OBS to AWS Elemental MediaLive


## v1.1.0 update highlights

Version **v1.1.0** keeps the two guides aligned as companion documents and adds more explicit AWS setup guidance, especially around IAM users, groups, roles, service permissions, billing access, and MediaLive service-role setup.

The update also refreshes the source-verification date and clarifies that no Cloudflare-specific steps are currently included in the guides.

## Recommended initial commit message

```text
Add AWS live streaming guide variants
```


## Recommended v1.1.0 commit message

```text
Add explicit AWS IAM setup guidance to streaming guides
```

## Notes

AWS service names, Console workflows, pricing, limits, and supported features can change over time. Each guide includes a last-verified date and source links for important AWS and OBS references.

Always verify current AWS pricing and service behavior before running long live streams or leaving AWS resources active.
