# homelab

Bare-metal Kubernetes homelab running on three repurposed mini PCs. Debian, no desktop, production-grade patterns.

## Hardware

| Host | Role | CPU | RAM | Storage |
|---|---|---|---|---|
| kserver | k3s master | Intel Celeron N2807 2c | 8GB, DDR3 | 212GB SSD |
| knode1 | k3s worker | Intel Celeron J4105 4c | 4GB, DDR4 | 13GB eMMC |
| knode2 | k3s worker | Intel Celeron J4105 4c | 4GB, DDR4 | 13GB eMMC |
| kbrain | LLM node | Intel Celeron J4105 4c | 8GB, DDR4 | 110GB SSD |
| kbrain2 | LLM node | AMD GX-415GA SOC 4c | 16GB, DDR3 | 56GB SSD |

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
| `node-firewalld` | Opens k3s API, kubelet, and flannel VXLAN ports |
| `k3s-master` | Installs k3s server (schedulable — runs pods like any other node) |
| `k3s-agent` | Joins worker nodes to the cluster |
| `k3s-labels` | Labels every node with `hostname=<hostname>` |
| `k3s-argocd` | Installs ArgoCD, ArgoCD ingress route |
| `k3s-gateway` | Gateway API CRDs, Envoy Gateway, GatewayClass |
| `node-maintenance` | Systemd timer — cleans up old container images and logs |
| `node-ollama-build` | Weekly systemd timer — builds fresh llama.cpp binaries on ainodes, keeps last N timestamped builds + a `latest` symlink |
| `node-ollama-model` | On-demand — downloads `ollama_model_url` (set per-group in `group_vars/aisnodes.yml` / `group_vars/aimnodes.yml`) into `/var/lib/llama-models`, removes the old model file |
| `otelcol-contrib` | OpenTelemetry Collector — ships host metrics and logs to the cluster |
| `ollama` | LLM inference on kbrain via Ollama |

## Playbooks

| Playbook | Targets | What it runs |
|---|---|---|
| `site.yml` | all | Full cluster bootstrap |
| `node-firewalld.yml` | master + nodes | Opens k3s API, kubelet, and flannel VXLAN ports |
| `k3s-master.yml` | master | Installs k3s server |
| `k3s-agent.yml` | nodes | Joins worker nodes to the cluster |
| `k3s-labels.yml` | master | Labels every node with `hostname=<hostname>` |
| `k3s-argocd.yml` | master | Installs ArgoCD |
| `k3s-gateway.yml` | master | Gateway API, Envoy Gateway |
| `node-maintenance.yml` | all | Node maintenance role |
| `node-ollama-build.yml` | ainodes | Weekly llama.cpp build role |
| `node-ollama-model.yml` | aisnodes + aimnodes | Downloads each group's model (see `group_vars/`), removes the old one |
| `brain.yml` | kbrain | otelcol + Ollama |

## Bootstrap

One-time OS setup per machine — static IP, SSH hardening, passwordless sudo:

```bash
nmcli con add type ethernet con-name lab-static ifname enp1s0 \
  ipv4.method manual ipv4.addresses 192.168.178.50/24 \
  ipv4.gateway 192.168.178.1 ipv4.dns "192.168.178.1 1.1.1.1"

# Hostname
sudo hostnamectl set-hostname <hostname>

# /etc/network/interfaces — static IP (Debian net-tools)
auto enp1s0
iface enp1s0 inet static
    address 192.168.178.10x
    netmask 255.255.255.0
    gateway 192.168.178.1
    dns-nameservers 8.8.8.8 8.8.4.4

# /etc/NetworkManager/system-connections/lab-static.nmconnection — static IP (NetworkManager)
[connection]
id=lab-static
type=ethernet
interface-name=enp1s0

[ipv4]
method=manual
address1=192.168.178.x/24,192.168.178.1
dns=192.168.178.1;1.1.1.1;

[ipv6]
method=auto

# SSH key auth only
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# Passwordless sudo
sudo visudo
# dmytro ALL=(ALL) NOPASSWD:ALL
```

Then run Ansible from your local machine:

```bash
# Full cluster bootstrap
ansible-playbook -i ansible/inventory.ini ansible/playbooks/site.yml

# k3s only, step by step
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s-master.yml
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s-agent.yml
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s-labels.yml
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s-argocd.yml
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s-gateway.yml

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
