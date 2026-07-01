# Changelog

All notable changes to this repository will be documented in this file.

This project follows semantic versioning-style release labels where practical.

## [v1.1.0] - Quick-start and IAM guidance update

### Added

- Added quick-start companion guide for the Amazon IVS path:
  - `aws-live-streaming-quick-start-guide-obs-ivs-v1.1.0.md`
  - Condenses the full IVS guide into a shorter, sequential setup path.
  - Points readers back to the full IVS guide for IAM setup, deeper explanations, troubleshooting, security, cleanup, and appendixes.

- Added quick-start companion guide for the AWS Elemental MediaLive path:
  - `aws-live-streaming-quick-start-guide-obs-medialive-v1.1.0.md`
  - Condenses the full MediaLive guide into a shorter, sequential setup path.
  - Clearly calls out that the MediaLive route is more advanced than the IVS route.
  - Points readers back to the full MediaLive guide for IAM/service-role setup, deeper explanations, troubleshooting, security, cleanup, and appendixes.

- Added one-page cheat sheet for the Amazon IVS path:
  - `aws-live-streaming-cheat-sheet-obs-ivs-v1.1.0.md`

- Added one-page cheat sheet for the AWS Elemental MediaLive path:
  - `aws-live-streaming-cheat-sheet-obs-medialive-v1.1.0.md`

- Added explicit AWS IAM/access setup guidance to the full guides:
  - Step 0 in each full guide
  - Appendix M in each full guide
  - IAM Identity Center/federated access guidance
  - IAM user/group fallback guidance for personal lab accounts
  - MFA, billing access, optional CLI access-key, and managed-policy guidance

- Updated README navigation:
  - Added a “Which file should I open first?” section.
  - Added quick-start guide and cheat-sheet links.
  - Clarified when to use quick starts, full guides, and cheat sheets.

### Changed

- Updated current repository version to `v1.1.0`.
- Updated the full guide metadata to v1.1.0.
- Updated last-verified notes in the full guides.
- Corrected the MediaLive RTMP Push start order so OBS starts pushing before the MediaLive channel starts.

### Notes

- The v1.1.0 quick-start guides and cheat sheets are based on the v1.1.0 full guides.
- The IVS path remains the simpler route.
- The MediaLive path remains the more advanced AWS media workflow.
- Recommended commit message:
  - `Add quick-start guides and cheat sheets`

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
