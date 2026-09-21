# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] - 2026-09-21

### Added

- Optional `status_entity`, in a new Status section: an `input_select` the
  automation sets to the mode it just applied — `Comfort`, `Comfort Boosted`,
  `Eco`, `Away`, `Frost Protection`, `Window Open` or `Off` — so a dashboard can
  show why the heating is doing what it is doing without re-deriving the whole
  priority chain in a template.

  Leaving it empty keeps the previous behaviour exactly; nothing is written and
  no helper is needed. The mode is computed in the same action-level
  `variables:` step as `target_temp`, with the same branch order plus the two
  HVAC-off cases in front, so the reported mode cannot disagree with the
  commanded one. It is written last, after the thermostats, and only when it
  differs from the helper's current state, so a steady state leaves no logbook
  entry. The call is `continue_on_error`, so a helper missing an option keeps
  its previous value instead of failing the run — the heating is already
  commanded by that point.

  Tradeoff: the option strings are fixed and case-sensitive. An `input_text`
  would not need them to match, but gives no known option set to style or
  filter on in a dashboard.

## [1.2.0] - 2026-09-20

### Added

- `heating_off_override` accepts multiple entities instead of one. They are
  OR-ed: heating is off while *any* of them is on. A room can then be turned
  off independently by, say, a load-shedding automation and a manual toggle,
  rather than having them share one boolean and overwrite each other.

  Existing automations are unaffected and need no edit: a stored single entity
  id still works, because `expand()` accepts a bare entity id as well as a
  list. The input key and its selector domains are unchanged, so this is
  additive, not breaking. Re-import the blueprint to pick it up; only then does
  the UI offer more than one entity.

## [1.1.0] - 2026-09-19

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
