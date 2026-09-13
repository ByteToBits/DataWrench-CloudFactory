## CloudFactory Ansible

Configuration management for the DataWrench MES fleet.

### Prerequisites

**Control node** runs on WSL2 (Ubuntu) or the CF-5 Ansible VM.

WSL requires mirrored networking to reach the lab VLANs. In `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
```

Then `wsl --shutdown` from PowerShell and reopen. Without this, WSL sits behind NAT and cannot route to the MES VLANs.

Keep the repo inside the WSL filesystem (`~/`), not `/mnt/c`. DrvFs reports world-writable permissions, which causes Ansible to silently ignore `ansible.cfg`.

**SSH key** — `~/.ssh/cf_homelab` (mode 600). The public half is installed to `cloudfactoryadmin` on all managed hosts. This key is the primary access path; back it up somewhere outside this machine.

**`ansible.cfg` is gitignored.** Copy the example and set your values:

```bash
cp ansible.cfg.example ansible.cfg
```

***

### Setup

```bash
cd CloudFactory/infra/Ansible
uv sync
uv run ansible-galaxy install -r requirements.yml
uv run ansible all -m ping
```

### Inventory

| Group | Hosts | VLAN | OS |
|---|---|---|---|
| `k8s_control_plane` | cf-mes-cp-01 | 101 | RHEL 10 |
| `k8s_workers` | cf-mes-wrk-01 | 101 | RHEL 10 |
| `data_services` | cf-mes-cache-01, cf-mes-db-01 | 102, 103 | Ubuntu / RHEL |
| `equipment` | cf-lcs-01 | 150 | Ubuntu |

Hosts also belong to an OS group (`rhel` or `ubuntu`) for OS-specific variables.

***

### Playbooks

Run all commands from `CloudFactory/infra/Ansible`.

#### System update

Full package upgrade across the fleet. Handles both `dnf` and `apt`.

```bash
uv run ansible-playbook playbooks/update_sys.yml
```

Runs `serial: 1` to avoid taking the whole cluster down at once. Reports reboot requirement but does not reboot.

#### NTP

Installs chrony, masks `systemd-timesyncd` on Debian hosts, points each host at its own VLAN gateway (always `.1`, derived from `ansible_host`).

```bash
uv run ansible-playbook playbooks/ntp.yml
```

Verify:

```bash
uv run ansible all -m shell -a "chronyc tracking | grep -E 'Reference ID|Leap status'" -b
```

Expect `Leap status: Normal` on every host.

#### SSH hardening

Deploys `/etc/ssh/sshd_config.d/10-hardening.conf`. Root login disabled, password auth enabled. The copy task validates with `sshd -t` before writing, so a bad config cannot lock out the fleet.

```bash
uv run ansible-playbook playbooks/hardening.yml
```

Verify:

```bash
uv run ansible all -m shell -a "sshd -T | grep -E 'permitrootlogin|passwordauthentication'" -b
```

#### Kubernetes Node Software

Installs CRI-O and the Kubernetes packages on `k8s_cluster`. Does **not** run `kubeadm init` or `join` — those are post-clone steps.

```bash
uv run ansible-playbook playbooks/k8s_node.yml
```

Single host:

```bash
uv run ansible-playbook playbooks/k8s_node.yml --limit cf-mes-cp-01
```

Verify:

```bash
uv run ansible k8s_cluster -m shell -a "kubeadm version -o short; crio --version | head -1" -b
uv run ansible k8s_cluster -m shell -a "crictl --runtime-endpoint unix:///var/run/crio/crio.sock images" -b
```

**Requires the QEMU Guest Agent enabled on the VM** (Proxmox → Options → QEMU Guest Agent). The VM needs a full stop/start afterwards — a reboot does not add the virtio device.


### PostgreSQL, Patroni and etcd

Installs the database stack on `patroni` hosts. Does **not** run `initdb`, bootstrap etcd, or start any service — those are post-clone steps.

```bash
uv run ansible-playbook playbooks/postgres_patroni.yml
```

Verify:

```bash
uv run ansible patroni -m shell -a "
systemctl is-enabled postgresql-18 patroni etcd 2>&1;
ls -la /var/lib/etcd/;
ls -la /var/lib/pgsql/18/data
" -b
```

Expect all three services `disabled`, `/var/lib/etcd` empty, and `/var/lib/pgsql/18/data` present but empty with mode `0700`. Patroni owns the data directory and runs `initdb` itself at bootstrap.

Requires EPEL for `python3-click`, which RHEL 10 does not ship. The role installs it. EPEL is community-maintained and broad — for a production deployment, restrict it with `includepkgs=python3-click`.

etcd is not packaged for RHEL 10 in any repo, so it installs from the upstream GitHub tarball with a hand-written systemd unit.

***

### Redis and Sentinel

Installs Redis and Sentinel on `cache` hosts from Redis's official apt repository. Both services are installed, stopped, and disabled — Sentinel topology is configured post-clone.

```bash
uv run ansible-playbook playbooks/redis_sentinel.yml
```

Verify:

```bash
uv run ansible cache -m shell -a "
systemctl is-enabled redis-server redis-sentinel 2>&1;
ls -la /var/lib/redis/;
apt-mark showhold
" -b
```

Expect both services `disabled`, no `dump.rdb` or `appendonlydir` in `/var/lib/redis`, and both packages held.

Ubuntu's package starts Redis on install and writes `dump.rdb`. The role removes it — that file must not exist in the golden image. Sentinel also rewrites its own config at runtime with discovered topology, so `/etc/redis/sentinel.conf` is state rather than configuration and is templated post-clone.

Ubuntu ships Redis in `universe` and Valkey in `main`. This role uses Redis's official repo instead, for upstream maintenance rather than community best-effort backports.

### Sealing a VM for cloning

Strips host identity so clones don't collide. **Destructive** — run only against the VM you intend to template.

Before sealing:

1. Apply all pending updates and reboot
2. Shut down the VM cleanly
3. Take a Proxmox snapshot (`pre-seal`) as a rollback point
4. Boot it back up

Put exactly one host in the `template_source` group in `inventory/hosts.yml`:

```yaml
    template_source:
      hosts:
        cf-mes-cp-01:
```

Then:

```bash
uv run ansible-playbook playbooks/seal.yml -e confirm_seal=yes
```

The playbook refuses to run without `confirm_seal=yes`, and refuses if `template_source` contains more than one host. Both guards exist because a mis-targeted run would strip identity from every VM at once.

After it completes, shut down and **do not boot the VM again** — booting regenerates machine-id and SSH host keys, undoing the seal. Convert to a Proxmox template, or `vzdump` it for transfer to another node.

Seal each VM role separately, editing `template_source` between each.

#### What sealing clears

| Item | Reason |
|---|---|
| `/etc/machine-id`, `/var/lib/dbus/machine-id` | DHCP identifier — clones would request the same lease |
| SSH host keys | clones would present identical fingerprints |
| cloud-init state | without a clean, cloud-init won't re-run and the clone keeps the source hostname |
| `/etc/hostname` | set per-clone by `personalize.yml` |
| logs, package caches, shell history | dead weight in every clone |

It also asserts that no cluster state exists (`/etc/kubernetes/pki`, `/var/lib/etcd/member`, `/var/lib/redis/dump.rdb`) and fails if it finds any. Cloning a VM that has already joined a cluster produces nodes with duplicate certificates and identities.

#### Red Hat subscriptions

Sealing does **not** unregister from Red Hat. The clone therefore boots carrying the source's consumer certificate and believes it is already registered. Each clone must drop the inherited certificates before registering itself:

```bash
sudo subscription-manager clean
sudo subscription-manager register --activationkey=<key> --org=<org-id>
```

Run `clean` first — registering on top of inherited certificates either fails or creates a duplicate entry.

Because the source is never unregistered, its entry remains in the Red Hat portal after the VM is gone. These accumulate across rebuilds and need periodic manual cleanup. Watch the system count if the subscription has a limit — a Developer Subscription allows 16.

Use an activation key rather than username and password so the command is non-interactive and no credentials sit in the playbook. Create one under Subscriptions → Activation Keys in the customer portal.

***

### Pinned versions

| Component | Version | Source |
|---|---|---|
| Kubernetes | 1.37.0 | `pkgs.k8s.io/core:/stable:/v1.37` |
| CRI-O | 1.36.5 | `download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.36` |
| PostgreSQL | 18.6-4PGDG.rhel10.2 | PGDG |
| Patroni | 4.1.5-1PGDG.rhel10.2 | PGDG (`patroni-etcd`) |
| etcd | 3.7.1 | GitHub release tarball |
| Redis / Sentinel | 6:8.10.1-1rl1~resolute1 | `packages.redis.io/deb` |
| RHEL | 10.2 | Red Hat CDN |
| Ubuntu | resolute | Ubuntu archive |

CRI-O intentionally trails Kubernetes by one minor — this is the upstream-documented pairing, not a workaround.

Kubernetes and CRI-O are pinned by `exclude` in their repo definitions; install tasks pass `disable_excludes` to bypass deliberately. PostgreSQL, Patroni and Redis are pinned by explicit version string, with Redis additionally held via `dpkg_selections`.

Patroni's etcd cluster is **separate from the Kubernetes control plane's etcd**. Do not point one at the other — a cluster rebuild would take the database's coordination layer with it.


### Conventions

Before committing:

```bash
uv run ansible-lint playbooks/<name>.yml
uv run ansible-playbook playbooks/<name>.yml --syntax-check
```

Test against one host before the fleet:

```bash
uv run ansible-playbook playbooks/<name>.yml --check --diff --limit <host>
uv run ansible-playbook playbooks/<name>.yml --limit <host>
```

Re-run to confirm idempotency — the second run should report `changed=0`.

