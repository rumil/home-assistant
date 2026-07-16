# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Home Assistant blueprint-publishing repo**, not an HA config directory. It tracks a single automation blueprint, `blueprints/switch-light-sync.yaml`. The live HA instance does not read this repo directly — it imports the blueprint from the file's `source_url` (the raw GitHub URL on `main`). There is no application to build, and no test/lint suite; the deliverable is declarative YAML validated by Home Assistant at import/instantiation time.

## Deploy loop (how a change reaches live HA)

1. Edit `blueprints/switch-light-sync.yaml`, commit, push to `main`.
2. In the HA UI: **Settings → Automations & Scenes → Blueprints →** the blueprint **→ ⋮ → Re-import blueprint**.

Hard constraints (do not rediscover the hard way):
- The Home Assistant MCP `ha_import_blueprint` tool **cannot overwrite** an existing blueprint, and there is no blueprint delete/override tool. The re-import is therefore a **manual UI step** — it cannot be automated from here.
- GitHub's raw CDN caches ~5 minutes; if a re-import doesn't reflect the latest push, wait and re-import again.
- After re-importing a changed blueprint, existing automation instances that use it must be re-validated. Adding a **required** input breaks every existing instance until each is updated — prefer adding new inputs with a `default:` so re-import stays non-breaking.

## Validating changes

- `python3` here has no `yaml` module. For a quick local syntax check use Ruby (it must tolerate the `!input` tag):
  ```bash
  ruby -ryaml -e 'YAML.add_domain_type("","input"){|t,v| {"input"=>v}}; YAML.load_file("blueprints/switch-light-sync.yaml"); puts "OK"'
  ```
- Real validation is HA-side. Use the Home Assistant MCP: `ha_eval_template` to test Jinja snippets against live entity state before embedding them, and `ha_get_blueprint` / `ha_config_get_automation` to confirm the live blueprint and instances after a re-import.

## Debugging a live misfire

`ha_get_automation_traces` keeps only **~5 runs per automation**, so traces from an incident an hour old are already gone. Reconstruct from the recorder instead — it is usually enough to identify the exact branch:

- `ha_get_history` on the light, the switch, **and** the instance's `input_text` store, `order="asc"`. The store's transitions show what each scenario decided to persist.
- `ha_get_logs(source="logbook", compact=False)` on the switch/light — `context_user_id` distinguishes a **user action** from a **device report** (firmware boot: no context at all) from an **automation's own call** (`context_domain: automation`). The boot heuristic keys on exactly this, so it is the fastest way to confirm which path ran.
- Timing is diagnostic: the gap from trigger to a scenario's first service call fingerprints the branch (e.g. B's boot path is `1s settle + turn_on + 2s` ⇒ its Grenton sync lands ~3.4 s after the trigger).
- `ha_get_logs(source="system")` — a burst of `<name>: Already running` entries means triggers are being dropped; cross-check against the desync window rather than assuming they are benign. The blueprint sets `max_exceeded: debug`, so these **no longer appear at WARNING** — an outage or restart makes `mode: single` drop dozens of self-inflicted re-entrant triggers per instance, which is by design and drowned the log. To see them, raise the level first: `logger.set_level` on `homeassistant.components.automation.<object_id>` = `debug`, then read with `level="DEBUG"`.
- Recorder/logbook timestamps come back in **local time** (Europe/Warsaw) while trace timestamps are **UTC** — do not compare them without converting.

## Blueprint architecture (`switch-light-sync.yaml`)

Two-way sync between a **Grenton virtual switch** and a **light** entity, resilient to restarts and power outages. `mode: single` — this is load-bearing: it drops the re-entrant triggers caused by the blueprint's own service calls, so scenarios can act on the light without fighting themselves.

`mode: single` cuts both ways, and the cost is not theoretical. A running scenario holds the only slot, so **any** state change landing mid-run is dropped (`execution: failed_single` in the trace) — including one that was needed to correct the state. There is no periodic reconcile, so a desync created this way persists until the next outage or user action. This makes the settle invariant below a correctness requirement, not a nicety: a scenario that reads stale state and drives an entity wrong has no second chance to fix it.

**Triggers** (each tagged with an `id`, dispatched by a top-level `choose`):
- `grenton_changed` / `grenton_recovered` — Grenton switch real change vs. return from unknown/unavailable.
- `light_changed` — light on↔off (real→real).
- `light_recovered` — light returns from unknown/unavailable (outage recovery).
- `light_low` — `attribute: brightness` trigger; catches a firmware boot that drops to ~1% **without** any state-string change (these lights sometimes report 1% without going unavailable). State triggers do not fire on attribute-only changes, hence the dedicated trigger.
- `ha_start` — HA finished starting.

**Scenario map** (the `choose` branches):
- **A (grenton_changed):** drive the light to match the switch, then save a snapshot.
- **B (light_changed OR light_low):** recompute from *current* state so the outcome is identical whichever of those **two triggers** wins under `mode: single` (both can fire for one on/off event). Note this only makes B indifferent to *which trigger started it* — it does **not** make B safe against its own service calls racing the state machine; see the settle invariant. Detects a firmware boot as "dropped into the low zone" (fresh drop, not a low→low repeat — that guard prevents a wake/restore loop) with no user context → wake to 100% + restore snapshot. Otherwise sync + save.
- **C (grenton_recovered OR ha_start):** on `ha_start`, if the light is already on at ≤threshold brightness, wake + restore (fixes a stuck-at-1% that persisted through an HA-only restart). Then sync the switch to the light (light is source of truth).
- **D (light_recovered):** the main outage-recovery path — settle, optionally wait for boot, wake if low, restore snapshot, sync.

**Persistence model:** the "saved state" is a JSON snapshot written to a per-instance `input_text` helper (`state_store` input) via `input_text.set_value`, and read back with `from_json`. This is deliberate: an HA automation **cannot** create a persistent scene (`scene.create` is wiped on restart) or create helpers at runtime, so the store helper is provisioned separately and passed in as an input. `saved_attributes` (input) selects which light attributes to snapshot; the save loop skips any the light reports as `none`. Saves never persist a device-reported low brightness or a non-on/off state (snapshot-poisoning guards).

## Blueprint conventions / gotchas

- **Settle after every service call before reading the state it changes.** `light.turn_on`/`turn_off` return as soon as the command is dispatched; the entity keeps reporting its *old* state until the device round-trips (MQTT/Zigbee: ~100–500 ms). A `condition: state` evaluated straight after a light command reads the pre-command value. Every restore→sync path therefore ends `restore → delay 1s → choose (sync)`; scenarios C and D always had this, and B's boot path silently lacked it until 2026-07-16 (fixed in `836978b`), which drove the Grenton switch to the state the light was *leaving* and — because `mode: single` then dropped the correcting `light_changed` — stranded a switch `on` against a light `off` for 2h10m. **If you add a branch that commands an entity and then reads it, settle in between.** The same reasoning applies to `light_low`'s brightness read (the 1s at the top of B) — Zigbee groups report `state: on` first and the `brightness` attribute ~500 ms later, so reading immediately sees `brightness: 0` and can misfire the boot heuristic.
- Target lights are **Zigbee2MQTT groups** (`platform: mqtt`, `unique_id: <n>_light_zigbee2mqtt`) — Z2M publishes the group as its own MQTT light and computes group state itself. The `group_entities` attribute is Z2M metadata, not an HA group-helper member list, so those member entity_ids may not resolve in HA (they are deliberately disabled on the live instance) and carry no recorder history. This does not affect the group's behaviour; it only means there is no per-fixture trail to debug with.
- **Optional `entity` inputs must be referenced via a template, not `!input`.** An unset entity input defaults to `""`; if `!input` puts that empty string into a service `target: entity_id:`, HA fails config-load validation for *every* instance (checked at load, regardless of runtime guards). Use `entity_id: "{{ some_var }}"` and gate the call on a `has_store`-style guard instead. See `state_store` for the pattern.
- Bind inputs to `variables:` and use the variable inside Jinja — `!input` is a YAML tag, not a template value.
- Light color attributes use `color_temp_kelvin` (mireds-based `color_temp` was removed in HA 2026.3).
- Keep `source_url` stable — it is the re-import anchor.
