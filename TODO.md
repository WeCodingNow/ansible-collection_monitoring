# TODO

- [ ] etcd-based service discovery: a Python "registrator" that registers and
  deregisters Prometheus targets in etcd under `/monitoring/<hostname>/<service>`,
  started per relevant service as a systemd templated unit instance, plus a
  Python service that subscribes to that prefix and updates Prometheus targets
  accordingly (`file_sd_configs`, most likely). Preferably event-based rather
  than long-polling, to avoid pointless wakeups. Carried over from
  `collection-dev/TODO.md`'s `alandreev.monitoring` sketch — this collection
  currently only supports static, inventory-declared targets
  (`monitoring_prometheus_jobs`).

- [ ] Alerting: Alertmanager as a fifth role, plus a way to declare alerting
  rules alongside `monitoring_prometheus_jobs`.

- [ ] Grafana via `apt.grafana.com` as a lighter alternative to the ~363 MB
  upstream tarball — trades the control-node cache for a per-managed-node apt
  fetch, worth it once more than a couple of monitoring hosts exist.

- [ ] TLS and real authentication for Prometheus/Grafana. Both roles currently
  assume a trusted network, matching `tcs-1x/main/monitoring/docker-compose.yml`'s
  own anonymous-Admin defaults.

# DONE
