# Changelog

All notable changes to this repository will be documented in this file.

This project follows semantic versioning-style release labels where practical.

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
