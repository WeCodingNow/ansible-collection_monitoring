# alandreev.monitoring

Deploys a Prometheus + Grafana monitoring stack, plus `node_exporter` and
`process-exporter`, from upstream release tarballs — native binaries managed
by systemd, no containers involved.

## Requirements

- `ansible-core` 2.15 or newer.
- Passwordless sudo on the managed nodes — every role runs unprivileged and
  opts into `become: true` only on the tasks that write to `/usr/local/bin`,
  `/etc`, `/var/lib` and systemd.
- Debian or EL hosts with systemd. Binaries are `linux-amd64` upstream
  release tarballs; no distro package manager is involved.

## Privilege model

Tasks run as the SSH login user by default. `become: true` is opted into only
where a task genuinely needs root: installing binaries into
`/usr/local/bin`, writing `/etc/<service>/` and `/var/lib/<service>/`, and all
systemd unit management. Service accounts are dedicated system users
(`nologin` shell): `prometheus`, `grafana`, `node_exporter`. `process_exporter`
is the one exception and runs **as root** — reading every process's
`/proc/<pid>` entry (not just the login user's own) is the whole point of the
exporter.

## The `common` role

Not meant to be used directly. `prometheus`, `node_exporter` and
`process_exporter` share one release shape — a GitHub tarball
`<name>-<version>.linux-amd64.tar.gz` with one top-level directory, alongside a
`checksums.txt` in `<sha256>  <filename>` format — so the fetch/checksum/
transfer/unpack/install sequence is factored into
`roles/common/tasks/install_release.yml`, included via:

```yaml
- name: install prometheus
  ansible.builtin.include_role:
    name: alandreev.monitoring.common
    tasks_from: install_release.yml
  vars:
    common_release_name: prometheus
    common_release_version: "{{ monitoring_prometheus_version }}"
    common_release_url: "https://github.com/prometheus/prometheus/releases/download/v{{ monitoring_prometheus_version }}/prometheus-{{ monitoring_prometheus_version }}.linux-amd64.tar.gz"
    common_release_checksum_url: "https://github.com/prometheus/prometheus/releases/download/v{{ monitoring_prometheus_version }}/sha256sums.txt"
    common_release_cache: "{{ monitoring_cache_dir }}/prometheus"
    common_release_archive_root: "prometheus-{{ monitoring_prometheus_version }}.linux-amd64"
    common_release_binaries: [prometheus, promtool]
```

Task sequence: ensure the control-node cache directory exists → fetch the
checksums file and extract this release's hash (delegated to `localhost`,
`run_once: true`) → `get_url` the tarball with `checksum: sha256:…` (which is
both the integrity check and the re-run no-op) → on the managed node, run
`<first binary> --version` and skip everything else if it already reports
`common_release_version` → `ansible.posix.synchronize` the cached tarball to
`/opt/monitoring-dist` → `unarchive` → `copy remote_src: true` each of
`common_release_binaries` into `/usr/local/bin`, mode `0755` (`become`).

Grafana does not use this — it ships from `dl.grafana.com`, not GitHub
releases, has a `.sha256` sidecar in a different format, and installs a whole
application directory rather than a couple of binaries. It caches on the
control node the same way, in its own `tasks/fetch.yml`.

## Roles

### `prometheus`

| Variable | Default | Meaning |
|---|---|---|
| `monitoring_prometheus_version` | `3.13.2` | Release to install |
| `monitoring_prometheus_listen_address` | `0.0.0.0:9090` | `--web.listen-address` |
| `monitoring_prometheus_external_url` | *(unset)* | `--web.external-url`, when Prometheus sits behind a reverse proxy |
| `monitoring_prometheus_retention_time` | `15d` | `--storage.tsdb.retention.time` |
| `monitoring_prometheus_scrape_interval` | `15s` | `global.scrape_interval` |
| `monitoring_prometheus_evaluation_interval` | `15s` | `global.evaluation_interval` |
| `monitoring_prometheus_global` | `{}` | merged into the rendered config's `global:` section, for anything else |
| `monitoring_prometheus_jobs` | `[]` | scrape jobs — see *Prometheus targets* below |
| `monitoring_prometheus_extra_scrape_configs` | `[]` | verbatim `scrape_configs` entries, appended after the ones built from `monitoring_prometheus_jobs` |
| `monitoring_prometheus_extra_args` | `[]` | extra flags appended to `ExecStart` |
| `monitoring_cache_dir` | `{{ inventory_dir }}/dist` | control-node download cache |

#### Prometheus targets

```yaml
monitoring_prometheus_jobs:
  - job_name: tcs-storage-tarantool          # required
    targets:                                 # required: list of "addr:port" strings
      - "{{ hostvars['s1-r01-i01'].int_ip }}:{{ hostvars['s1-r01-i01'].roles_cfg['roles.metrics-export'].http[0].listen }}"
      - 10.0.1.18:8888
    labels: {stand: tcs-test}                 # optional -> static_configs[0].labels
    metrics_path: /metrics                    # any other scrape_config key,
    scrape_interval: 5s                        # copied through verbatim
```

`targets` and `labels` become the job's single `static_configs` entry; every
other key in a job (`metrics_path`, `scheme`, `params`, `basic_auth`,
`relabel_configs`, …) is copied into the rendered `scrape_config` unchanged, so
nothing beyond `targets`/`labels` needs dedicated role support. Each target is
a complete address string — ports are per-target and never inferred, and are
free to be any Jinja expression over `hostvars`/`groups`, which is what lets a
job reference a port declared on some other host in the same inventory rather
than a value hardcoded here. Jobs that need real service discovery (not a
static target list) belong in `monitoring_prometheus_extra_scrape_configs`
instead, appended verbatim.

The rendered `/etc/prometheus/prometheus.yml` is validated with `promtool
check config` before it's written — a malformed config fails the task instead
of leaving Prometheus running on stale config, or worse, dead. Config changes
notify a **reload** (not restart) handler, so a target change doesn't churn
the TSDB.

Tags: `download` (the control-node fetch), `prometheus`, `monitoring-config`
(re-render the config and reload, without reinstalling the binaries).

```bash
ansible-playbook -i inventory.yml site.yml --tags monitoring-config
```

### `grafana`

| Variable | Default | Meaning |
|---|---|---|
| `monitoring_grafana_version` | `13.1.1` | Release to install |
| `monitoring_grafana_listen_port` | `3000` | `[server] http_port` |
| `monitoring_grafana_admin_password` | `admin` | `[security] admin_password` |
| `monitoring_grafana_anonymous_enabled` | `true` | `[auth.anonymous] enabled` |
| `monitoring_grafana_anonymous_org_role` | `Admin` | `[auth.anonymous] org_role` |
| `monitoring_grafana_basic_auth_enabled` | `false` | `[auth.basic] enabled` |
| `monitoring_grafana_ini` | `{}` | extra `grafana.ini` sections, merged as `{section: {key: value}}` |
| `monitoring_grafana_prometheus_url` | `http://127.0.0.1:9090` | URL of the bundled default datasource (see below), and available to user-supplied datasource files |
| `monitoring_grafana_datasources` | `[]` | control-node paths to datasource provisioning files — see *Dashboards and datasources* |
| `monitoring_grafana_dashboards` | `[]` | `[{src, folder}]` — see *Dashboards and datasources* |
| `monitoring_cache_dir` | `{{ inventory_dir }}/dist` | control-node download cache |

The anonymous-Admin, basic-auth-off defaults reproduce
`tcs-1x/main/monitoring/docker-compose.yml`'s current Grafana environment, so
a stand migrating off compose sees the same first-visit behavior.

#### Dashboards and datasources

```yaml
monitoring_grafana_dashboards:
  - src: "{{ inventory_dir }}/../../monitoring/grafana/application/provisioning/dashboards"
    folder: TCS
  - src: ~/my-dashboards/latency.json
    folder: Ad-hoc

monitoring_grafana_datasources:
  - "{{ inventory_dir }}/../../monitoring/grafana/application/provisioning/datasources"
  - ~/my-datasources/loki.yaml
```

Both are control-node paths — a directory (copied recursively) or a single
file. A directory of datasources is restricted to `*.yaml`/`*.yml`, matching
what Grafana's datasource provisioner reads — a directory may otherwise also
hold unrelated exports (e.g. a raw datasource JSON dumped from Grafana's own
API), which this skips rather than pushing at Grafana as if it were a
provisioning file. `monitoring_grafana_dashboards` entries need a `folder`,
since Grafana's file-based dashboard provisioning wants one provider per
folder (`foldersFromFilesStructure` is deliberately left off — an explicit
folder is what the `{src, folder}` shape promises, and mixing the two would be
ambiguous); provider names are derived from the folder and must be unique —
a duplicate folder across two entries fails loudly rather than letting Grafana
silently drop a provider. `monitoring_grafana_datasources` is flatter: Grafana
datasource files declare their own `name`/`uid`, so there's no folder to carry.
Datasource files are rendered with `ansible.builtin.template`, so one may
reference `monitoring_grafana_prometheus_url` or any other variable in scope.

**Datasources travel with the dashboards that need them.** A dashboard JSON
that hardcodes a panel's datasource `uid`, and the provisioning file that
declares that same `uid`, are one artifact — ship them from the same `src`
directory rather than trying to keep a role-level uid variable in sync with
vendored dashboard JSON by hand. `monitoring_grafana_datasources` left empty
falls back to one bundled Prometheus datasource at
`monitoring_grafana_prometheus_url`, marked `isDefault: true`, so the role
still stands up a working Grafana on its own; supplying the variable replaces
that default outright (no merging with it).

Dashboard file changes within an already-configured folder are picked up by
Grafana's own polling (`updateIntervalSeconds: 10` in the generated
providers) — but the providers file itself, which declares which folders
exist, is only read at startup, so a new folder needs a restart before it
shows up in the UI. Datasource provisioning does not poll at all — Grafana
only reloads it on restart — so every datasource change, a new dashboard
folder, and `grafana.ini` changes all notify a restart handler.

Tags: `download`, `grafana`, `monitoring-config`, plus `grafana-dashboards` for
pushing dashboard/datasource files alone.

### `process_exporter`

| Variable | Default | Meaning |
|---|---|---|
| `monitoring_process_exporter_version` | `0.8.7` | Release to install |
| `monitoring_process_exporter_listen_address` | `0.0.0.0:9256` | `-web.listen-address` |
| `monitoring_process_exporter_threads` | `true` | whether to pass `-threads` |
| `monitoring_process_exporter_config` | `{process_names: [{comm: [k6]}]}` | the exporter's own config, written verbatim |
| `monitoring_process_exporter_extra_args` | `[]` | extra flags |
| `monitoring_cache_dir` | `{{ inventory_dir }}/dist` | control-node download cache |

`monitoring_process_exporter_config` is a plain dict, rendered straight to
YAML — whatever `process-exporter`'s own config schema accepts (matching by
`comm`, `cmdline`, `exe`, grouping rules, …) can be written here with no
per-key role support. The default reproduces
`tcs-1x/main/monitoring/process-exporter/config.yml`. Config changes notify a
restart handler (`process-exporter` has no reload signal).

Runs **as root** — see *Privilege model* above.

Tags: `download`, `process-exporter`, `monitoring-config`.

### `node_exporter`

| Variable | Default | Meaning |
|---|---|---|
| `monitoring_node_exporter_version` | `1.12.1` | Release to install |
| `monitoring_node_exporter_listen_address` | `0.0.0.0:9100` | `--web.listen-address` |
| `monitoring_node_exporter_collectors_enabled` | `[]` | each becomes `--collector.<name>` |
| `monitoring_node_exporter_collectors_disabled` | `[]` | each becomes `--no-collector.<name>` |
| `monitoring_node_exporter_extra_args` | `[]` | extra flags |
| `monitoring_cache_dir` | `{{ inventory_dir }}/dist` | control-node download cache |

Tags: `download`, `node-exporter`, `monitoring-config`.

## The playbook

`alandreev.monitoring.site` in two plays, each with an overridable host
pattern (an imported play's `hosts:` cannot be overridden any other way, and
`--limit` would clamp every play in the run):

```yaml
- ansible.builtin.import_playbook: alandreev.monitoring.site
  vars:
    monitoring_exporter_hosts: dev_vms   # node_exporter + process_exporter
    monitoring_server_hosts: monitoring  # prometheus + grafana
```

`monitoring_exporter_hosts` defaults to `all`, `monitoring_server_hosts` to a
`monitoring` group. Aim `monitoring_exporter_hosts` at the physical-node group
whenever several inventory hosts (e.g. several Tarantool instances) share one
machine — pointing it at a service group instead would run the exporter roles
once per instance on the same box.

## Example

```yaml
# inventory.yml
monitoring:
  vars:
    monitoring_prometheus_jobs:
      - job_name: node
        targets: ["10.0.0.28:9100"]
    monitoring_grafana_dashboards:
      - src: ~/my-dashboards/
        folder: Default
  hosts:
    monitoring-1:
```

```yaml
# site.yml
- ansible.builtin.import_playbook: alandreev.monitoring.site
```

## License

MIT
