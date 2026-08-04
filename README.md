# ansible-monitoring

An Ansible **collection**, `alandreev.monitoring`, that deploys a Prometheus +
Grafana monitoring stack — plus `node_exporter` and `process-exporter` — as
native binaries managed by systemd. This repo is the collection's source tree;
it isn't a one-off playbook repo.

## What's here

```
ansible_collections/alandreev/monitoring/   the collection itself (galaxy.yml, roles/, playbooks/)
TODO.md                                     known follow-up work
```

Targets **`ansible==8.5.0`** (ansible-core 2.15.13) exactly, and this repo's
own venv is pinned to **Python 3.13** (see `pyproject.toml`) — the same
constraint `alandreev.dev` and `tcs-deploy` share, since a stand that
provisions all three runs one interpreter.

## Roles

| Role | What it does | Configurable via |
|------|---------------|-------------------|
| `prometheus` | Prometheus + `promtool` from an upstream release tarball, config rendered from an inventory-declared job list, systemd-managed | `monitoring_prometheus_jobs`, `monitoring_prometheus_extra_scrape_configs`, `monitoring_prometheus_version`, `monitoring_prometheus_listen_address`, `monitoring_prometheus_retention_time`, `monitoring_prometheus_scrape_interval` / `_evaluation_interval`, `monitoring_prometheus_global`, `monitoring_prometheus_external_url`, `monitoring_prometheus_extra_args` |
| `grafana` | Grafana from an upstream release tarball, dashboards and datasources provisioned from control-node files, systemd-managed | `monitoring_grafana_dashboards`, `monitoring_grafana_datasources`, `monitoring_grafana_version`, `monitoring_grafana_listen_port`, `monitoring_grafana_admin_password`, `monitoring_grafana_anonymous_enabled` / `_org_role`, `monitoring_grafana_basic_auth_enabled`, `monitoring_grafana_ini`, `monitoring_grafana_prometheus_url` |
| `process_exporter` | `process-exporter` from an upstream release tarball, running as root (it reads every process's `/proc/<pid>`), config supplied verbatim | `monitoring_process_exporter_config`, `monitoring_process_exporter_version`, `monitoring_process_exporter_listen_address`, `monitoring_process_exporter_threads`, `monitoring_process_exporter_extra_args` |
| `node_exporter` | `node_exporter` from an upstream release tarball, running as its own unprivileged user | `monitoring_node_exporter_version`, `monitoring_node_exporter_listen_address`, `monitoring_node_exporter_collectors_enabled` / `_disabled`, `monitoring_node_exporter_extra_args` |

All four install into `/usr/local/bin`, write their config under `/etc/<name>/`,
and run as a systemd service. `prometheus` and `grafana` also share a
control-node artifact cache (see below); `prometheus`, `node_exporter` and
`process_exporter` additionally share the fetch/checksum/install task
sequence, factored into an internal `common` role
(`roles/common/tasks/install_release.yml`) — the one piece that genuinely is
identical across them, the same reasoning `alandreev.dev`'s `common` role
follows for its `dev_vm_<x>_config` precedence logic.

## Prometheus targets

`monitoring_prometheus_jobs` is a list of Prometheus `scrape_config` entries,
with two keys treated specially:

```yaml
monitoring_prometheus_jobs:
  - job_name: tcs-storage-tarantool
    targets:
      - "{{ hostvars['s1-r01-i01'].int_ip }}:{{ hostvars['s1-r01-i01'].roles_cfg['roles.metrics-export'].http[0].listen }}"
      - 10.0.1.18:8888
    labels: {stand: tcs-test}
    metrics_path: /metrics
```

`targets` (required) and `labels` (optional) become the job's single
`static_configs` entry; every other key is copied into the `scrape_config`
verbatim, so `params`, `relabel_configs`, `basic_auth` and the rest need no
role-specific support. Each target is a full `"addr:port"` string — ports are
never assumed uniform across a job, and are free to be built with Jinja from
other hosts' inventory data (as in the Tarantool example above), which is what
lets this role reference services declared elsewhere in the same inventory.
Jobs needing real service discovery (rather than a static target list) go in
`monitoring_prometheus_extra_scrape_configs`, appended verbatim.

## Grafana dashboards and datasources

```yaml
monitoring_grafana_dashboards:
  - src: ~/my-dashboards/           # a directory, copied recursively, or a single .json
    folder: TCS
  - src: ~/my-dashboards/latency.json
    folder: Ad-hoc

monitoring_grafana_datasources:
  - ~/my-datasources/               # a directory of provisioning files, or a single one
  - ~/my-datasources/loki.yaml
```

Both are lists of paths **on the control node**. `monitoring_grafana_dashboards`
needs a `folder` per entry (Grafana's file provisioning wants one provider per
folder); `monitoring_grafana_datasources` is flatter, since Grafana datasource
provisioning files declare their own name/uid and don't nest into folders.
Datasource files are templated on the way over, so one may reference
`monitoring_grafana_prometheus_url` or any other inventory variable. Left
empty, `grafana` provisions one bundled Prometheus datasource at
`monitoring_grafana_prometheus_url`, marked default, so the collection stands
up a working Grafana on its own — supplying the variable replaces that
default outright, no merging.

A dashboard JSON that pins a specific datasource `uid` and the provisioning
file that declares that same `uid` are one artifact: the provisioning file's
`uid` has to keep matching whatever the vendored dashboard JSON hardcodes,
rather than trying to keep an inventory-supplied uid variable in sync with it
by hand. Usually that means sourcing both from the same directory; if the
datasource file also has to stay usable somewhere Jinja won't reach it (a
docker-compose bind mount, say), a separate templated copy of just the
datasource file works too, as long as its `uid` doesn't drift from the one the
dashboards were vendored with.

## Control-node artifact cache

`prometheus`, `grafana`, `node_exporter` and `process_exporter` each fetch
their release tarball once onto the control node (delegated `localhost`
tasks, checksum-guarded so a re-run with an unchanged version is a no-op),
then transfer it to the managed node — provisioning a second host reuses the
cache instead of re-downloading. This matters most for `grafana`: its tarball
is roughly 363 MB.

All four read `monitoring_cache_dir`, which defaults to
`{{ inventory_dir }}/dist` — the cache sits next to the inventory of whatever
stand is being provisioned, the same convention `role-etcd` and
`alandreev.dev` use.

## The playbook

The collection ships one playbook, `alandreev.monitoring.site`, split into two
plays so exporters and the Prometheus/Grafana pair can be aimed at different
host patterns — an imported play's `hosts:` cannot be overridden any other
way, and `--limit` would clamp every play in the run:

```yaml
- ansible.builtin.import_playbook: alandreev.monitoring.site
  vars:
    monitoring_exporter_hosts: dev_vms   # every physical node to run exporters on
    monitoring_server_hosts: monitoring  # where Prometheus + Grafana live
```

This split matters whenever several inventory hosts (e.g. Tarantool instances)
share one physical machine: aiming the exporter play at a service group would
redo the same work once per instance, so it targets the physical-node group
instead.

## How to use

### Prerequisites

- A Python 3.13 environment with `ansible==8.5.0` installed (this repo's own
  `.venv`, via `uv sync` / `pip install -e .`, already satisfies this).
- SSH access to the target host(s) with a sudo-capable user, passwordless sudo.
- Debian or EL hosts with systemd — the roles install upstream `linux-amd64`
  release tarballs directly, no distro package manager involved.

`ansible.cfg` at the repo root already sets `collections_path = .`, so
`ansible-playbook`/`ansible-galaxy` resolve `alandreev.monitoring.*` correctly
as long as you run from the repo root — no `ansible-galaxy collection
install` step needed for local use.

### Example inventory

```yaml
monitoring:
  vars:
    monitoring_prometheus_jobs:
      - job_name: node
        targets: ["10.0.1.8:9100"]
    monitoring_grafana_dashboards:
      - src: ~/my-dashboards/
        folder: Default
  hosts:
    monitoring-1:
```

A working example against a real VM is checked in at `.local/inventory.yml`
(gitignored — specific to this machine's SSH setup).

### Running it

```bash
# Sanity-check without touching anything
ansible-playbook alandreev.monitoring.site -i inventory.yml --syntax-check

# Provision (or re-provision — every role is idempotent)
ansible-playbook alandreev.monitoring.site -i inventory.yml

# Re-render Prometheus' config and reload, without reinstalling
ansible-playbook alandreev.monitoring.site -i inventory.yml --tags monitoring-config
```

## Design docs

`ansible_collections/alandreev/monitoring/README.md` documents each role's
variables in full; this file covers the collection as a whole.
