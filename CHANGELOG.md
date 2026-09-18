# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed

- `boost_preset` was never mapped into `variables`, so every run logged
  `'boost_preset' is undefined` and, with preset modes on, no preset was set
  during boost
- Boost preset was not applied during scheduled comfort periods: that branch
  used `comfort_preset` while the matching temperature branch used the boosted
  value
- Leaving for a named zone ("Work") did not count as leaving: the away and
  frost protection triggers only matched `not_home`. Moving between zones while
  away also no longer pushes the frost protection time back.
- An unavailable climate entity ended the update loop, skipping every entity
  after it
- A failing thermostat call (cloud API error) aborted the run for the remaining
  entities; climate calls now continue on error
- A manual "Run actions" read `trigger.id` on a trigger that has none
- The availability trigger also fired on `unknown` -> `unavailable`
- `preset_modes` reported as `None` was not treated as an empty list
- An unparseable frost protection helper (`unknown`, or a time-only helper)
  made the run raise; it is now treated as nothing scheduled
- The window/door selector offered every binary sensor: `domain` and
  `device_class` were two separate, OR-ed filters

### Changed

- Migrated to `triggers:`/`conditions:`/`actions:` and `action:`, which requires
  Home Assistant 2024.10 or newer, now declared with `homeassistant.min_version`
- Added `source_url` so the blueprint can be updated in place after import
- Target temperature, preset and HVAC mode are computed once per run instead of
  once per climate entity
- Use `has_value()` for climate availability
- The target temperature no longer falls back to 0 if it fails to render; the
  run stops instead of sending 0°C to the thermostats
- Docs: automatic frost protection requires the datetime helper, window
  detection waits for the configured duration, and both overrides force HVAC
  off regardless of order. The blueprint description and README said otherwise.

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
