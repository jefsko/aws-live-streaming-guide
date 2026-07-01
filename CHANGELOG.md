# Changelog

All notable changes to this repository will be documented in this file.

This project follows semantic versioning-style release labels where practical.

## [v1.1.0] - AWS access and synchronization update

### Added

- Added explicit AWS access, IAM, role, and permissions setup guidance to both companion guides.
- Added `Step 0` sections before the main AWS setup walkthroughs:
  - IVS guide: AWS access, IAM, and permissions setup.
  - MediaLive guide: AWS access, IAM, roles, and permissions setup.
- Added Appendix M to both guides:
  - IVS guide: AWS access, IAM, and permissions setup.
  - MediaLive guide: AWS access, IAM, roles, and permissions setup.
- Added clearer guidance for IAM users, IAM groups, managed policies, billing access, MFA, CLI access keys, and least-privilege follow-up.
- Added official AWS source links for IAM user/group setup, IVS managed policies, MediaLive managed policies, MediaLive service-role setup, MediaPackage v2 managed policies, CloudFront managed policies, and AWS Billing access.

### Changed

- Updated both guide versions to `v1.1.0`.
- Updated both guide `Last verified` dates to July 1, 2026.
- Updated prerequisites in both guides to point readers to the new explicit AWS access setup instructions.
- Updated quick-start paths in both guides to tell readers to confirm AWS access and permissions before creating resources.
- Updated the MediaLive guide to start OBS streaming before starting the MediaLive channel for the RTMP Push workflow, matching AWS MediaLive guidance that a push input should be pushing before the channel starts.
- Updated README current version to `v1.1.0`.

### Verified

- Re-checked official AWS and OBS source links used by both guides.
- Confirmed no Cloudflare-specific source links or Cloudflare-specific implementation steps are currently present in the guides.
- Verified both guides retain the same companion-guide structure while allowing guide-specific IVS and MediaLive details.
- Verified Markdown footnote references and definitions are balanced in both updated guides.

### Notes

- This is an additive companion-guide update. The core guide structure and v1.0.0 content were preserved while adding explicit AWS setup detail and correcting the MediaLive RTMP Push start-order guidance.


## [v1.0.0] - Initial release

### Added

- Added the Amazon IVS guide:
  - `aws-live-streaming-guide-obs-ivs.md`
  - Covers OBS Studio on Windows 11 streaming to Amazon IVS for public browser playback.
  - Focuses on the simpler AWS-native live streaming path.
  - Includes prerequisites, quick-start path, setup steps, billing notes, troubleshooting, security notes, cleanup steps, glossary, index, and appendixes.

- Added the AWS Elemental MediaLive guide:
  - `aws-live-streaming-guide-obs-medialive.md`
  - Covers OBS Studio on Windows 11 streaming to AWS Elemental MediaLive.
  - Uses a more advanced AWS media workflow with MediaLive, MediaPackage v2, and CloudFront for browser playback.
  - Clearly calls out the added complexity and cost considerations compared with the IVS path.
  - Includes prerequisites, quick-start path, setup steps, billing notes, troubleshooting, security notes, cleanup steps, glossary, index, and appendixes.

- Added repository documentation:
  - `README.md`
  - `CHANGELOG.md`

### Notes

- Initial release version: `v1.0.0`
- Recommended initial commit message:
  - `Add AWS live streaming guide variants`
- Initial guide filenames:
  - `aws-live-streaming-guide-obs-ivs.md`
  - `aws-live-streaming-guide-obs-medialive.md`
