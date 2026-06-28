# SPEC — Semconv Normalization + Per-Service `service.name`

> **Status:** Spec (pre-implementation)
> **Slug:** `otelcol`
> **Scope for commits:** `otelcol`
> **Extends:** the log/metric/resource pipeline in [SPEC.md](SPEC.md) §4.3

## 1. Objective

The bridge is a transport; the real producers are Home Assistant Core, the
Supervisor, and each add-on container (including the otelcol add-on itself).
Make the emitted telemetry reflect that and follow OpenTelemetry semantic
conventions:

1. **Every signal is attributed to its source service** via `service.name` —
   HA Core → `homeassistant`, each add-on → its slug, Supervisor →
   `supervisor`, the otelcol add-on (bridge stats, self-metric, **and** the
   collector's own internal telemetry) → its install slug.
2. **`service.version`** carries the producing add-on's version.
3. **Semconv record attributes** — rename HA log attributes with standard
   equivalents (`ha.logger` → `code.namespace`, `ha.source` → `code.filepath`).
4. **`ha.addon.version` removed** globally; `service.version` replaces it.

**Target user:** the LGTM-stack user (SPEC.md §1) who wants per-service debug
views — pivot on one `service.name` to see an add-on's logs *and* resource
metrics together — without losing fleet-wide comparison.

## 2. Decisions (locked during Q&A)

| Topic | Decision |
|---|---|
| Mechanism | otelcol **`groupbyattrs` + `transform`** (not Python multi-provider) |
| `service.name` value | **slug**, mapping `core` → `homeassistant`, `supervisor` → `supervisor` |
| Logs | per-source `service.name` |
| Metrics | per-source `service.name` for `supervisor.addon.*`; HA Core metrics → `homeassistant` |
| Traces | **no split** — no per-add-on identity; all → `homeassistant` |
| Bridge self-metric | tagged with the otelcol add-on slug → its own service |
| Collector internal telemetry | relabeled `otelcol-contrib` → otelcol add-on slug |
| Collector internals → backend | **deferred to Phase 2** (relabel now, export later, done properly) |
| `ha.addon.version` | **removed globally** |
| Packaging | **single always-on behavior**, no `config.yaml` option |
| Rollout | **breaking change** — migration note required |

### Non-Goals
- Splitting **traces** by source (no add-on identity in HA event graphs).
- Renaming HA-specific attributes without a semconv equivalent
  (`ha.event_type`, `ha.domain`, `ha.service`, `ha.context_id`, …) — keep `ha.*`.
- A config toggle.

## 3. Why `groupbyattrs`

`service.name` is a **resource** attribute, but the Python SDK fixes one
resource per signal-type provider, so source identity (`addon.slug`) lives on
each **record/datapoint**. A `transform` alone can't split a shared resource —
last writer wins. `groupbyattrs` regroups records/datapoints sharing a key
value into **separate** `ResourceLogs` / `ResourceMetrics`; only then does
`transform` stamp each resource. Order: **group, then transform** — same for
logs and metrics.

Records/datapoints **without** `addon.slug` fall through to the base resource
(`service.name=homeassistant`) — exactly where genuine HA Core data
(`ha.state.*`, `ha.entity.count`, `ha.events.total`, HA event traces) belongs.

## 4. Resource attribution model

| Signal | Carries `addon.slug`? | `service.name` |
|---|---|---|
| Container logs (add-ons) | yes (`extra`) | the add-on slug |
| HA Core file logs (`/core/logs`) | yes (`core`) | `homeassistant` (mapped) |
| `system_log_event` / lifecycle logs | yes (`core`, added) | `homeassistant` (mapped) |
| Supervisor logs | yes (`supervisor`) | `supervisor` |
| `supervisor.addon.*` stats | yes (per slug) | the add-on slug (`homeassistant` for `core`) |
| `ha.bridge.context_lru_size` | yes (`_SELF_SLUG`, added) | otelcol add-on slug |
| Collector internal telemetry (`:8888`) | n/a (relabeled) | otelcol add-on slug |
| `ha.state.*`, `ha.entity.count`, `ha.events.total` | no | `homeassistant` (base) |
| HA event traces | no | `homeassistant` (base) |
| External OTLP traces (4317/4318) | n/a | sender's own resource — passes through untouched |

> External OTLP traces are a capability, not a default: the ports are `null`
> in `config.yaml`, so traces only arrive if a user points an instrumented app
> at this collector. `traces/bridge` gets **no** resource manipulation, so the
> bridge's HA-event spans become `homeassistant` and any external spans keep
> their sender's `service.name` — doing nothing is what preserves them.

Fleet view is preserved: `service.name` becomes a label (`job`) in
Mimir/Prometheus, so `supervisor_addon_cpu_percent` grouped by `service_name`
still compares all add-ons; `{service_name="X"}` gives the per-add-on debug
view. Series count is unchanged (identity moves from `addon.slug` to
`service.name`).

## 5. Pipeline design

| Pipeline | Processors |
|---|---|
| `logs/bridge` (backend) | `memory_limiter, transform/semconv_logs, groupbyattrs/addon, transform/service_resource, batch` |
| `logs/debug` (panel) | `memory_limiter, transform/semconv_logs, filter/severity, batch` |
| `metrics/bridge` (backend) | `memory_limiter, groupbyattrs/addon, transform/service_resource, batch` |
| `traces/bridge` | `memory_limiter, batch` |

- `transform/semconv_logs` (record renames) is on **both** log pipelines so
  panel and backend agree; emitted regardless of endpoint.
- `groupbyattrs/addon` + `transform/service_resource` are emitted **only when
  `otlp_endpoint` is set**. With no endpoint: `logs/bridge` is absent and
  `metrics/bridge` is `[memory_limiter, batch]`.
- The old `resource` processor (only attribute: `ha.addon.version`) is
  **removed from every pipeline**.

### 5.1 Generated processor YAML

```yaml
processors:
  transform/semconv_logs:
    log_statements:
      - context: log
        statements:
          - set(attributes["code.namespace"], attributes["ha.logger"]) where attributes["ha.logger"] != nil
          - delete_key(attributes, "ha.logger")
          - set(attributes["code.filepath"], attributes["ha.source"]) where attributes["ha.source"] != nil
          - delete_key(attributes, "ha.source")
          - set(attributes["ha.log.count"], attributes["ha.count"]) where attributes["ha.count"] != nil
          - delete_key(attributes, "ha.count")
          - set(attributes["ha.log.first_occurred"], attributes["ha.first_occurred"]) where attributes["ha.first_occurred"] != nil
          - delete_key(attributes, "ha.first_occurred")

  groupbyattrs/addon:
    keys: [addon.slug]

  # One processor, two signal blocks. addon.slug is at resource level after
  # groupbyattrs; addon.version/name remain per record/datapoint (consistent
  # within a single-slug group), so lift them with the !=nil guard.
  transform/service_resource:
    log_statements:
      - context: log
        statements:
          - set(resource.attributes["service.name"], resource.attributes["addon.slug"]) where resource.attributes["addon.slug"] != nil
          - set(resource.attributes["service.name"], "homeassistant") where resource.attributes["addon.slug"] == "core"
          - set(resource.attributes["service.name"], "supervisor") where resource.attributes["addon.slug"] == "supervisor"
          - set(resource.attributes["service.version"], attributes["addon.version"]) where attributes["addon.version"] != nil
          - set(resource.attributes["addon.name"], attributes["addon.name"]) where attributes["addon.name"] != nil
    metric_statements:
      - context: resource
        statements:
          - set(attributes["service.name"], attributes["addon.slug"]) where attributes["addon.slug"] != nil
          - set(attributes["service.name"], "homeassistant") where attributes["addon.slug"] == "core"
          - set(attributes["service.name"], "supervisor") where attributes["addon.slug"] == "supervisor"
      - context: datapoint
        statements:
          - set(resource.attributes["service.version"], attributes["addon.version"]) where attributes["addon.version"] != nil
```

`groupbyattrs` keys on **`addon.slug` only** so HA Core's two log sources (file
logs + `system_log_event`, differing in `addon.name`/version) collapse into one
`homeassistant` resource. Both processors ship in `otelcol-contrib` 0.154.0.

### 5.2 Collector internal-telemetry relabel

The collector's own internal telemetry resource defaults to
`service.name=otelcol-contrib`. Override it to the install slug so the otelcol
add-on is one coherent service. The `run` script already emits
`service.telemetry.logs.level`; add a sibling `resource` key:

```yaml
service:
  telemetry:
    resource:
      service.name: "<install-slug>"   # e.g. 03f32180_otelcol
    logs:
      level: ${otelcol_log_level}
```

The slug is derived in bash like the bridge's `_SELF_SLUG`:
`slug=$(hostname | tr '-' '_')`. This flows through `:8888` `target_info` to the
`prometheus/internal` scrape, relabeling the internal metrics (and the
collector's own internal logs). Emitted **regardless of endpoint** (internals
are debug-only today — see Phase 2).

> **Trade-off:** standard Grafana otelcol dashboards key on
> `service.name=otelcol-contrib` / `job=otel-collector` and will need their
> selector pointed at the install slug. Accepted for one-service consistency.

## 6. Bridge changes (`rootfs/usr/bin/ha-otel-bridge`)

The bridge only emits the attributes the processors consume — no new provider,
no per-source resource management, no record renames.

1. **Base resource**: `service.name=homeassistant`,
   `service.namespace=home-assistant`.
2. **Capture add-on versions** in `_enumerate_services` — extract
   `addon.get("version")` (currently discarded), carry it in the service tuple,
   and populate `_addon_versions[slug] = version` for the stats callbacks.
3. **Container-log `extra`** (`_follow_logs`, `_poll_logs`): add `addon.version`.
4. **HA Core structured events**: add `addon.slug="core"` to the `extra` of
   `handle_system_log_event` and `handle_lifecycle_log`.
5. **Stats datapoints** (`_container_cpu_callback`, `_container_mem_callback`):
   add `addon.version` from `_addon_versions` alongside `addon.slug`.
6. **Bridge self-metric** (`_context_lru_size_callback`): emit with
   `{"addon.slug": _SELF_SLUG}`.

> The bridge's own stderr logger (`getLogger("ha-bridge")`) has no OTLP handler
> and never enters the pipeline; no tagging needed. The `_follow_logs`
> `_SELF_SLUG` skip (debug-exporter feedback-loop guard) **stays** — only the
> self-**metric** gains the slug tag.

## 7. Config-generation changes (`s6-rc.d/otelcol/run`)

1. Remove the `resource` processor block, `ha.addon.version`, and the unused
   `addon_version` / `bashio::addon.version` lookup. Remove `resource` from all
   pipeline processor lists.
2. Always emit `transform/semconv_logs`; add to both log pipelines.
3. When `otlp_endpoint` is set: emit `groupbyattrs/addon` +
   `transform/service_resource`; add both to `logs/bridge` and `metrics/bridge`
   (group **before** the transform). When unset: `metrics/bridge` is
   `[memory_limiter, batch]`.
4. Emit `service.telemetry.resource.service.name: <slug>` (§5.2), slug from
   `hostname | tr '-' '_'`.
5. Leave `filter/severity` in `logs/debug` and the `traces/bridge` list intact.

## 8. Before / After

**Before** — one service, collector version on everything:
```
ResourceMetrics { service.name: ha-otel-bridge, ha.addon.version: 0.3.10 }
  supervisor.addon.cpu_percent{addon.slug=d5369777_music_assistant} 0.07
  supervisor.addon.cpu_percent{addon.slug=03f32180_otelcol} 9.71
  ha.entity.count{ha.domain=light} 14
ResourceLogs { service.name: ha-otel-bridge }
  "...HA Core error..." { ha.logger: homeassistant.components.script }
debug/internal: { service.name: otelcol-contrib }   # collector internals
```

**After** — per-source services, semconv attrs, no collector version:
```
ResourceMetrics { service.name: d5369777_music_assistant, service.version: 2.x }
  supervisor.addon.cpu_percent 0.07
ResourceMetrics { service.name: 03f32180_otelcol }
  supervisor.addon.cpu_percent 9.71
  ha.bridge.context_lru_size 82
ResourceMetrics { service.name: homeassistant, service.namespace: home-assistant }
  ha.entity.count{ha.domain=light} 14
ResourceLogs { service.name: homeassistant }
  "...HA Core error..." { code.namespace: homeassistant.components.script }
debug/internal: { service.name: 03f32180_otelcol }   # relabeled (Phase 2: → backend)
```

## 9. Phases

1. **Phase 1 (this spec).** Semconv renames, per-source `service.name`/version
   for logs + metrics, base resource → `homeassistant`, bridge tagging,
   `ha.addon.version` removal, collector internal-telemetry **relabel** to slug.
   Internals remain debug-only.
2. **Phase 2 (follow-up — "collector 다음").** Export `metrics/internal` to the
   OTLP backend, implemented properly (the bundled-config exporter merge needs
   care — exporter slices replace, not append, on merge). Out of scope here; the
   relabel in Phase 1 makes internals land under the slug once exported.

## 10. Testing Strategy

No unit harness; verify via config validation + live smoke (repo preflight).

1. **Config validity** — `otelcol-contrib validate` on generated YAML (the
   `run` script already does this pre-exec): passes with the new processors, no
   `resource` processor, and `service.telemetry.resource` set — endpoint-set and
   endpoint-empty.
2. **Preflight** — `yamllint config.yaml translations/*.yaml`; `shellcheck -x run`.
3. **Live smoke (real `otlp_endpoint`)**:
   - Each source is a distinct `service.name` in Loki **and** Mimir;
     `service.version` populated where available.
   - `supervisor_addon_cpu_percent` grouped by `service_name` shows all add-ons
     (fleet view intact).
   - HA Core file logs + `system_log_event` both → `service.name=homeassistant`.
   - otelcol add-on's stats + `context_lru_size` share its slug service; the
     collector's debug/internal output now shows the slug (not
     `otelcol-contrib`); nothing bridge-own under `homeassistant`.
   - Migrated logs carry `code.namespace`/`code.filepath`; no `ha.logger` /
     `ha.source` / `ha.addon.version` anywhere.
   - Add-on **panel** still renders (severity filter unaffected).
4. **Inertness** — with `otlp_endpoint` empty: no `logs/bridge`, no
   `groupbyattrs`/`transform/service_resource`; `transform/semconv_logs` still
   on `logs/debug`; `metrics/bridge` is `[memory_limiter, batch]`; relabel still
   applied.

## 11. Rollout / Migration (breaking)

- **`service.name`** for container logs and `supervisor.addon.*` metrics moves
  from `ha-otel-bridge` to per-source values; collector internals move from
  `otelcol-contrib` to the install slug. Queries on
  `service.name="ha-otel-bridge"` / `"otelcol-contrib"` must move to the
  per-source name, or `service.namespace="home-assistant"` to match HA sources.
- **Renamed log attributes**: `ha.logger`→`code.namespace`,
  `ha.source`→`code.filepath`, `ha.count`→`ha.log.count`,
  `ha.first_occurred`→`ha.log.first_occurred`.
- **`ha.addon.version` removed**; use per-source `service.version`.
- **CHANGELOG**: headline as breaking. **DOCS.md**: document the resource model,
  `service.name` mapping, renames, and the collector-dashboard selector change.
  Release via [[app-dev-pr]].

## 12. Boundaries

### Always Do
- Group **before** transforming to a resource.
- Emit split processors only on the `otlp_endpoint`-set path; keep no-endpoint
  config inert (relabel excepted — it has no backend cost).
- Validate generated YAML before launch (existing `run` behavior).
- Keep separation/rename logic in otelcol, not Python (AGENTS.md).
- Preserve the `_SELF_SLUG` self-log skip (feedback-loop guard).

### Ask First
- Fetching HA Core / Supervisor versions for their `service.version` (extra
  `/core/info` / `/supervisor/info` calls).
- Phase 2 exporter wiring into the bundled self-monitoring config.
- Splitting **traces** by any HA attribute (no meaningful key today).

### Never Do
- Don't split `logs/debug` (panel stays as-is aside from shared renames).
- Don't rename `ha.*` attributes lacking a semconv equivalent.
- Don't reintroduce a global collector-version attribute under a non-semconv key.

## 13. Success Criteria

1. `otelcol-contrib validate` passes (new processors present, no `resource`
   processor, `service.telemetry.resource` set), endpoint-set and -empty.
2. `yamllint` + `shellcheck -x run` pass.
3. With a backend: each source is a distinct `service.name` across logs and
   metrics; `service.version` populated where available; fleet metric still
   groupable by `service_name`.
4. HA Core file logs and `system_log_event` share `service.name=homeassistant`;
   otelcol stats + `context_lru_size` + (Phase 2) internals share its slug.
5. Migrated logs carry `code.namespace`/`code.filepath`; no `ha.addon.version`
   anywhere.
6. With no `otlp_endpoint`, generated config differs from today only by the
   `resource`-processor removal, `transform/semconv_logs` on `logs/debug`, and
   the `service.telemetry.resource` relabel.
