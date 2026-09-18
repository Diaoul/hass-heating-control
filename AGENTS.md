# Working in this repo

## What this is

One Home Assistant **blueprint** — `heating_control.yaml` — plus its docs.
There is no application here, no build, no test suite, no CI.

A blueprint is a parameterised automation template. Users import it, then create
automations *from* it, filling in `input:` fields through the Home Assistant UI.
The file declares `blueprint.input` (what users pick), `variables` (computed
values), `triggers`, `conditions` and `actions`.

Consequences that are easy to miss:

- **The file in this repo is not what runs.** Importing copies it to
  `config/blueprints/automation/<user>/heating_control.yaml` on the user's
  Home Assistant. Editing here changes nothing until they re-import. When
  debugging, confirm which version their install actually has.
- **A user's automation stores only the input values**, referencing the
  blueprint by path. So input keys are a public API: renaming one, or changing
  its selector type, breaks existing automations.
- **The import badge in `README.md` points at `main`**, not at a tag. Whatever
  is on `main` is what new users get, released or not.
- **It cannot be tested here.** Templates render only inside Home Assistant.
  Local checks catch YAML and Jinja mistakes; everything else is verified by the
  maintainer running it and exporting a trace.

## What it controls, and why that matters

Room heating: one automation per room, driving one or more `climate` entities
from a schedule, presence, windows, and manual overrides. The decision tables
in `README.md` (Advanced Documentation) are the spec; keep the code and the
tables in step.

- **Thermostats are often cloud-backed.** The maintainer's run through Overkiz,
  which is rate-limited and polled, so the entity state lags a command by some
  seconds. Every command is guarded by a comparison with the entity's current
  state, so a steady state sends nothing. Keep it that way: a change that sends
  commands unconditionally, or re-sends them before the state has caught up,
  floods the vendor API.
- **Frost protection is the safety output.** It exists so an empty house in
  winter doesn't freeze, and so a house left heated doesn't stay heated.
- **Presets and temperatures are exclusive.** With preset modes on, only
  presets are sent. Setting a temperature on many devices switches them into a
  manual/derogation mode, so don't "fall back" to a setpoint.

## How the automation actually works

Stateless reconcile. Every run recomputes the target HVAC mode, temperature
and preset from current entity states, then commands each climate entity only
where it differs. A missed run is harmless because the next one recomputes
from scratch.

The one exception is frost protection, whose state lives in the
`input_datetime` helper the user supplies: a time in the future is a
countdown, a time in the past means active, and `1970-01-01 00:00:00` means
nothing scheduled. A helper that doesn't parse as a date and time (`unknown`,
or a time-only helper) is treated as nothing scheduled. The trigger-specific
steps at the top of `actions:` write that helper; the targets are then
computed in an action-level `variables:` step so they see the value just
written. Top-level `variables:` render before
any action and would read the stale value.

- `target_temp` and `target_preset` are parallel `if`/`elif` chains. Each
  branch must pick the matching pair — boosted temperature with boosted
  preset — or presets and temperatures drift apart. Change them together.
- Every climate command has `continue_on_error: true`, so an API error on one
  entity doesn't skip the entities after it.
- Targets are computed from current state, not from what triggered the run.
  The `for:` on a window trigger only delays *that* trigger; any other run
  while a window is open turns the heating off straight away.
- Triggers are event-driven only: restart, a climate entity becoming available,
  and state changes of every input entity. There is no periodic tick, so a
  thermostat changed by hand stays changed until the next trigger.
- Presence triggers use `not_to: home` / `from: home` rather than
  `to: not_home`, because a person in a named zone has that zone as their state.
- `mode: single`. An overlapping run is dropped and logs `Already running`.
  `queued` and `restart` + debounce were considered and deliberately not
  adopted without data: `queued` replays a burst against stale read-back state
  and duplicates commands; a debounce is only useful if it outlasts the
  integration's state lag, which hasn't been measured. Measure before changing.
  `max_exceeded: silent` is not set so that the drops stay visible.

## Syntax: track current Home Assistant

Use the current forms, not the legacy ones that still happen to work.

- `triggers:` / `conditions:` / `actions:`, not the singular keys
- `action:` for service calls, not `service:`
- `trigger: <platform>` inside a trigger, not `platform:`
- Template condition shorthand: `- "{{ ... }}"` rather than
  `condition: template` + `value_template:`
- Purpose-built template functions over hand-rolled equivalents:
  `has_value(entity)` rather than comparing `states(entity)` against a list of
  `unavailable` / `unknown`
- Availability triggers pair `from: [unavailable, unknown]` with
  `not_to: [unavailable, unknown]`, or they fire on `unknown` -> `unavailable`
- Keep `blueprint.source_url`; it is what lets users update the blueprint in
  place after import

When a current form raises the minimum version, declare it rather than leaving
it implied:

```yaml
blueprint:
  homeassistant:
    min_version: "2024.10.0"
```

The plural keys need 2024.10. If you use a selector option or template function
added later, raise `min_version` to match — an undeclared requirement fails at
import with a confusing error instead of a clear one.

## Selectors and inputs

Prefer passing a `target:` selector straight to the action. `climate_target` is
the exception: it is an entity list, looped per entity, because each entity's
current state has to be compared before commanding it. The comment on the
`repeat:` says so; don't "simplify" it into a single target call.

- **A state trigger cannot take a target.** `entity_id:` needs entity ids, so
  an input using a target selector cannot also drive the availability trigger.
  Another reason `climate_target` stays an entity list.
- **Changing an input's selector type is breaking.** Home Assistant *merges* the
  stored value into the new shape rather than replacing it. A target schema
  rejects the stray keys and the automation refuses to load. Document it, bump
  the major version, and quote the exact error users will see.
- **Selector `filter:` is a list of OR-ed filters.** `domain` and
  `device_class` meant to apply together go in the *same* list item.
- Optional entity inputs default to `[]`, and every use is guarded with
  `| length > 0`.

## Template facts worth not rediscovering

- `!input` is not visible to templates. Every input a template uses must be
  mapped in `variables:` first. An unmapped one renders empty with only a
  `'x' is undefined` warning — `boost_preset` shipped that way.
- `variables:` render in order; each is available to the next, with
  `literal_eval` applied — a template rendering `[1, 2]` yields a real list.
- `trigger` is always defined. A manual "Run actions" supplies
  `{'platform': None}` — defined, but with no `id`. That is why `trigger_id`
  exists as a variable with `trigger.id | default('', true)`; use it rather
  than `trigger.id`.
- Manual runs skip top-level `conditions:` entirely. A guard there protects
  triggered runs only.
- A failing `condition` inside a `repeat` sequence ends the whole repeat, not
  the iteration. Use `if:`/`then:` to skip one item.
- `default()` replaces *undefined*, not `None`. `state_attr()` returns `None`
  for a missing attribute, so use `or []`.
- Jinja supports `**` unpacking, so `timedelta(**duration_input)` is fine.

## Don't invent fallbacks

A default that lets the automation keep acting on invented data is worse than
stopping. `target_temp | float(0)` would have sent 0°C to every thermostat if
the target ever failed to render; it is `| float` so the run stops instead.
`current_temp | float(0)` is fine: the worst case is one redundant command.

Likewise, don't add defensive handling for a failure mode you have only
imagined. Fix what is observed.

## Comments

Comments explain **why**. If a comment restates the line below it, delete it.

```yaml
# bad
# Calculate if schedule is active
schedule_active: >-

# good
# An offset of 0 means "use eco" rather than "same as comfort"
away_temp: >-
```

## Verify before committing

Templates are not type-checked and a broken blueprint fails at runtime,
unattended, in winter. Check what can be checked:

```bash
# YAML parses, with the Home Assistant tags registered
python3 -c "
import yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!input', lambda l,n: {'!input': l.construct_scalar(n)})
yaml.load(open('heating_control.yaml'), L); print('OK')"
```

For non-trivial Jinja, render it locally with `jinja2` and mocked HA functions
before committing, and check boundary cases, not just the middle: the frost
countdown at epoch, just before, exactly at, and just after `now()`; every row
of the README priority table.

State clearly what was verified and what was not. `expand()`, `has_value()`,
`is_state()` and friends only exist inside Home Assistant; local tests mock
them, and that is not the same as the code working.

## Debugging: traces are the source of truth

Ask for a trace export (Settings → Automations → Traces → download) rather than
reasoning from the config. The trace carries **Changed Variables**, with every
rendered value and type, plus the result of every condition and which branch
ran.

Read the numbers before proposing a fix, and check where a log line comes from
before attributing it to the blueprint. `value_json` warnings are MQTT entity
templates (Zigbee2MQTT), not this file.

## Docs must match the code

Three surfaces drift, in rising order of how often they are missed:

1. `README.md`, including the priority tables
2. `CHANGELOG.md`
3. the blueprint's own `description:` and each input's `description:` — what
   users read in the UI, and the ones that go stale

When behaviour changes, update all three in the same commit, including the
tradeoffs.

## Versioning and releases

Semantic versioning. A change requiring users to touch their existing
automation is a major bump, whatever its size. Record breaking changes under a
`### Breaking` heading at the top of the release, with the remedy and the exact
error.

Version lives in `README.md` and `CHANGELOG.md` only — blueprints have no
version field. Tag annotated, as `vX.Y.Z`.

## Commits

Commit messages explain the reasoning, not the diff: what was wrong, why the
chosen fix, and what tradeoff it accepts. Commits are gpg-signed; if signing
times out, retry rather than disabling it.
