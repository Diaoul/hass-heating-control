# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-12-30

### Added

- Initial release of Heating Control Blueprint
- Temperature control with comfort/eco modes
- Away overlay: applies configurable offset to comfort during scheduled periods (default -1°C, or eco fallback if offset is 0)
- Boost overlay: applies configurable offset to comfort for temporary temperature increase (supports schedule entities and binary sensors, default +1°C)
- Smart presence detection with person entities and sensors
- Schedule integration for defining comfort periods
- Window/door detection with configurable delays
- Automatic frost protection with manual override and datetime helper for persistence
- Guest mode support (acts as additional person entity)
- Optional preset mode support with dedicated presets for each mode
- Configurable durations for all state changes
