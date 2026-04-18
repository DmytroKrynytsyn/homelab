# homelab

Bare-metal Kubernetes homelab running on three repurposed mini PCs. Debian, no desktop, production-grade patterns.

## Hardware

| Host | Role | CPU | RAM | Storage |
|---|---|---|---|---|
| kserver | k3s master | Intel Celeron N2807 2c | 8GB | 212GB SSD |
| knode1 | k3s worker | Intel Celeron J4105 4c | 4GB | 13GB eMMC |
| knode2 | k3s worker | Intel Celeron J4105 4c | 4GB | 13GB eMMC |
| kbrain | LLM node | Intel Celeron J4105 4c | 8GB | SSD |

All machines run Debian bare-metal with static IPs on a home LAN (`192.168.178.x`).

## Cluster

Three-node k3s cluster managed by Ansible. ArgoCD handles all workload deployments via GitOps.

| Component | Role |
|---|---|
| k3s | Lightweight Kubernetes |
| ArgoCD | GitOps — App of Apps pattern |
| Envoy Gateway | Gateway API ingress — HTTP + HTTPS |
| cert-manager | Automatic TLS certificate provisioning |
| Flannel | CNI — VXLAN overlay |
| klipper-lb | Bare-metal LoadBalancer |

Storage-heavy workloads (VictoriaMetrics, VictoriaLogs) are pinned to `kserver` via `nodeSelector` — worker eMMC is too constrained.

## Ansible roles

| Role | What it does |
|---|---|
| `k3s-master` | Installs k3s server, ArgoCD, Gateway API CRDs, Envoy Gateway, GatewayClass |
| `k3s-agent` | Joins worker nodes to the cluster |
| `k3s-ufw` | Opens required ports — Kubernetes API, Kubelet, Flannel, HTTP, HTTPS |
| `node-maintenance` | Systemd timer — cleans up old container images and logs |
| `otelcol-contrib` | OpenTelemetry Collector — ships host metrics and logs to the cluster |
| `ollama` | LLM inference on kbrain via Ollama |

## Playbooks

| Playbook | Targets | What it runs |
|---|---|---|
| `site.yml` | all | Full cluster bootstrap |
| `k3s.yml` | master + nodes | k3s install and join |
| `common.yml` | all | UFW rules |
| `maintenance.yml` | all | Node maintenance role |
| `brain.yml` | kbrain | otelcol + Ollama |

## Bootstrap

One-time OS setup per machine — static IP, SSH hardening, passwordless sudo, UFW deny-all:

```bash
# /etc/network/interfaces — static IP
auto enp1s0
iface enp1s0 inet static
    address 192.168.178.10x
    netmask 255.255.255.0
    gateway 192.168.178.1
    dns-nameservers 8.8.8.8 8.8.4.4

# SSH key auth only
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# Firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22
sudo ufw enable

# Passwordless sudo
sudo visudo
# dmytro ALL=(ALL) NOPASSWD:ALL
```

Then run Ansible from your local machine:

```bash
# Full cluster bootstrap
ansible-playbook -i ansible/inventory.ini ansible/playbooks/site.yml

# k3s only
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s.yml

# LLM node
ansible-playbook -i ansible/inventory.ini ansible/playbooks/brain.yml
```

## Workloads

All deployed via ArgoCD. See individual repos:

- [infra-monitoring](https://github.com/DmytroKrynytsyn/infra-monitoring) — VictoriaMetrics, VictoriaLogs, Grafana, otelcol
- [agents-infra-test](https://github.com/DmytroKrynytsyn/agents-infra-test) — FastAPI microservice demo

## Design notes

**eMMC vs SSD placement** — storage-heavy components pinned to kserver via `nodeSelector`. Workers are compute-only.

**OS-layer maintenance below Kubernetes** — node cleanup handled via Ansible roles and systemd timers, not CronJobs.

**Inventory-driven conditionals** — hardware-specific tasks (smartctl on kserver only) driven by inventory variables, not playbook conditionals.

**Gateway API over Ingress** — Envoy Gateway with HTTPRoute replaces Traefik Ingress. TLS terminates at the Gateway level via a cert-manager provisioned wildcard cert.