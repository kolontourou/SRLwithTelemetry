# SRL with Telemetry

A Containerlab topology of a 5-node SR Linux CLOS fabric (2 spines + 3 leaves)
with three Linux hosts on VLAN 100 / VLAN 300 and a full telemetry stack
(gNMIc, Prometheus, Loki, Promtail, Grafana).

<p align="center">
  <a href="https://codespaces.new/kolontourou/SRLwithTelemetry?quickstart=1">
    <img alt="Open in GitHub Codespaces" src="https://github.com/codespaces/badge.svg">
  </a>
  <br>
  <em>Machine type: 4 vCPU · 16 GB RAM</em>
</p>

## Topology


| Node       | IP           | Role                                |
| ---------- | ------------ | ----------------------------------- |
| spine1     | 172.20.22.10 | SR Linux underlay spine             |
| spine2     | 172.20.22.11 | SR Linux underlay spine             |
| leaf1      | 172.20.22.12 | SR Linux leaf (host1 / VLAN 100)    |
| leaf2      | 172.20.22.13 | SR Linux leaf (host2 / VLAN 300)    |
| leaf3      | 172.20.22.14 | SR Linux leaf (host3 / VLAN 300)    |
| host1      | 172.20.22.15 | tenant client (VLAN 100, dnsmasq)   |
| host2      | 172.20.22.16 | tenant client (VLAN 300)            |
| host3      | 172.20.22.17 | tenant client (VLAN 300)            |
| gnmic      | 172.20.22.91 | gNMI collector                      |
| prometheus | 172.20.22.92 | TSDB (UI on `localhost:9090`)       |
| loki       | 172.20.22.94 | log aggregation                     |
| promtail   | 172.20.22.90 | syslog → Loki forwarder             |
| grafana    | 172.20.22.93 | dashboards (UI on `localhost:3000`) |


Tenant data plane: `host1@10.100.1.10` (VLAN 100) — `host2@10.100.3.10` and
`host3@10.100.3.11` (VLAN 300). Underlay uses /31 point-to-point links between
each leaf and both spines.

## Run locally

Prerequisites: Docker and [Containerlab](https://containerlab.dev/install/) on
the host.

```bash
sudo clab deploy -t topology.clab.yml
```

Once converged (~60–90 s), open Grafana at [http://localhost:3000](http://localhost:3000) (anonymous
admin login) and Prometheus at [http://localhost:9090](http://localhost:9090).

To tear down:

```bash
sudo clab destroy -t topology.clab.yml --cleanup
```

## Run in Codespaces

Click the button above. GitHub provisions a 4 vCPU / 16 GB cloud VM, the
Containerlab dev container starts, and you can deploy the same topology with
`sudo clab deploy -t topology.clab.yml`. Grafana and Prometheus ports are
forwarded automatically; the codespace UI offers a "Open in Browser" button
when the lab comes up.

> **Note**: Codespaces VMs do not support nested virtualization, so VM-based
> Containerlab kinds (vrnetlab / vSIM / SR-SIM) cannot run there. This lab is
> 100% container-based and works.

## Repository layout

```
.
├── topology.clab.yml                     # Containerlab topology
├── topology.clab.yml.annotations.json    # clab graph annotations / layout
├── leaf{1,2,3}.cfg, spine{1,2}.cfg       # SR Linux startup configs
├── front-panel-{leaf,spine}-single.svg   # Grafana flow-panel front panels
├── topology.svg                          # static topology diagram
├── configs/
│   ├── dns/                              # dnsmasq + /etc/hosts seed for host1
│   └── telemetry/
│       ├── gnmic/                        # gNMI subscriptions → Prometheus
│       ├── grafana/                      # datasources, provisioned dashboards, plugin
│       ├── loki/                         # Loki single-node config
│       ├── prometheus/                   # Prometheus scrape config
│       └── promtail/                     # syslog → Loki shipping
└── .devcontainer/devcontainer.json       # Codespaces / Dev Containers spec
```

## Credentials

- SR Linux nodes: `admin / NokiaSrl1!`
- Grafana: anonymous admin enabled (no login required)
- Hosts: `admin / multit00l`

## Notes

### Tenant DNS resolves over the data plane

`host1` runs `dnsmasq` (bound from `configs/dns/dnsmasq-host1.conf`) and serves
the lab's hostnames (`leaf1`, `host2`, …) on `10.100.1.10`. `host2` and `host3`
are configured with `nameserver 10.100.1.10` in `/etc/resolv.conf`, which sits
on the **tenant data plane**, not the management network. As a result:

- Hostname-based commands from `host2`/`host3` (e.g. `ping host1`,
`curl http://host1`) only succeed **after** the underlay (OSPF/BGP) and the
EVPN service between leaves have converged — typically 60–90 s after
`clab deploy`.
- Until then, IP-based commands work fine: `ping 10.100.1.10`,
`ping 172.20.22.15` (mgmt), etc.
- This is intentional — it exercises the fabric — but it can look like a DNS
outage on first boot. If `ping host1` fails, give the lab another minute or
use the IP directly.

### Grafana flow-panel plugin is unsigned

The topology view in Grafana is rendered by the
`[andrewbmchugh-flow-panel](https://github.com/andrewbmchugh/flow-panel)`
plugin, which is shipped vendored under
`configs/telemetry/grafana/plugins/andrewbmchugh-flow-panel/` (no Grafana
catalog download at runtime). It is **not signed** by Grafana Labs, so the
container loads it via the env var below:

```yaml
env:
  GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS: andrewbmchugh-flow-panel
```

If you copy this lab into a hardened Grafana environment (corporate Grafana,
production deployments, Grafana Cloud, etc.) the panel will refuse to load
without that env var. Either keep the env var, replace the panel with a signed
alternative (e.g. the built-in *Node Graph* or *Geomap* panels), or sign the
plugin yourself with a private signature.

## License

This repository (the topology, configs, dashboards and helper scripts) is
released under the **MIT License** — see the `LICENSE` file at the repo root.
You can fork, modify and redistribute it freely, just keep the copyright
notice.

### Third-party components

The repository vendors a third-party Grafana panel plugin which is **not**
covered by the MIT license:


| Component                                  | Version | License    | Upstream                                                                                                         |
| ------------------------------------------ | ------- | ---------- | ---------------------------------------------------------------------------------------------------------------- |
| `andrewbmchugh-flow-panel` (Andrew McHugh) | 1.20.1  | Apache-2.0 | [https://github.com/andymchugh/andrewbmchugh-flow-panel](https://github.com/andymchugh/andrewbmchugh-flow-panel) |


The plugin's upstream `LICENSE` is shipped alongside its binaries at
`configs/telemetry/grafana/plugins/andrewbmchugh-flow-panel/LICENSE` (Apache
License 2.0, Copyright © 2024 Andrew McHugh) to satisfy Apache-2.0 §4(a/c)
when redistributing it as part of this lab.
