## CloudFactory Ansible

Configuration management for the DataWrench MES fleet.

***

### Quick reference

Run from `CloudFactory/infra/Ansible`.

```bash
# Is everything up?
uv run ansible all -m ping -o
uv run ansible -i inventory/proxmox.yml proxmox -m ping -o

# Fleet identity check — hostname, address, MAC, netplan files
uv run ansible all -o -b -m shell -a 'echo "$(hostname) | $(ip -4 -br addr show ens18 | awk "{print \$3, \$4}") | $(cat /sys/class/net/ens18/address) | $(ls /etc/netplan 2>/dev/null | tr "\n" " ")"'

# Patroni cluster state — leader, replicas, lag
uv run ansible cf-mes-db-01 -b -m shell -a "sudo -u postgres patronictl -c /etc/patroni/patroni.yml list"

# Checkpoint the whole fleet (VMs stopped, snapshotted, restarted)
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_snapshot.yml -e snapshot=<name>

# Stop all VMs — hypervisors and firewall stay up
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_shutdown.yml -e confirm_shutdown=yes -e vms_only=true

# Start all VMs again (respects each node's start order)
uv run ansible -i inventory/proxmox.yml proxmox -m command -a 'pvenode startall'

# Full power-off: HYP-02 + HYP-03 → HYP-01 → FortiGate
# Needs physical access to power back on (the FortiGate 61F has no power button)
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_shutdown.yml -e confirm_shutdown=yes --ask-vault-pass
```

**Power-on order after a full shutdown:** FortiGate → HYP-01 → HYP-02 and HYP-03.

***

### Prerequisites

**Control node** runs on WSL2 (Ubuntu) or the CF-5 Ansible VM.

WSL requires mirrored networking to reach the lab VLANs. In `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
```

Then `wsl --shutdown` from PowerShell and reopen. Without this, WSL sits behind NAT and cannot route to the MES VLANs. The file must be named exactly `.wslconfig` — Notepad appends `.txt` by default.

Keep the repo inside the WSL filesystem (`~/`), not `/mnt/c`. DrvFs reports world-writable permissions, which causes Ansible to silently ignore `ansible.cfg`.

**SSH key** — `~/.ssh/cf_homelab` (mode 600). The public half is installed to `cloudfactoryadmin` on every VM and to `root` on every hypervisor. It is the primary access path; back it up somewhere outside this machine.

**Local files not in git** — copy each example and fill in real values:

| File | Example | Contains |
|---|---|---|
| `ansible.cfg` | `ansible.cfg.example` | remote user, key path |
| `inventory/group_vars/all.yml` | `all.example.yml` | user, key, become |
| `inventory/proxmox.yml` | `proxmox.example.yml` | hypervisor addresses and VM IDs |
| `vars/fortigate.vault.yml` | — | FortiGate address and API token (ansible-vault) |
| `vars/patroni.vault.yml` | `vars/patroni.example.yml` | PostgreSQL and Patroni passwords (ansible-vault) |

Set `private_key_file` in `ansible.cfg` as well as in `group_vars`. `group_vars` only loads with the real inventory; without the `ansible.cfg` setting, ad-hoc runs against a bare IP connect with no key and are refused.

***

### Setup

```bash
cd CloudFactory/infra/Ansible
uv sync
uv run ansible-galaxy install -r requirements.yml
uv run ansible all -m ping
```

***

### Inventory

Three of every role, one per hypervisor. VM IDs encode the node: `1xx` on HYP-01, `2xx` on HYP-02, `3xx` on HYP-03.

| Role | Group | -01 | -02 | -03 | VLAN | OS |
|---|---|---|---|---|---|---|
| Control plane | `k8s_control_plane` | .101.11 | .101.12 | .101.13 | 101 | RHEL 10 |
| Worker | `k8s_workers` | .101.101 | .101.102 | .101.103 | 101 | RHEL 10 |
| Cache | `cache` | .102.11 | .102.12 | .102.13 | 102 | Ubuntu |
| Database | `patroni` | .103.11 | .103.12 | .103.13 | 103 | RHEL 10 |
| Equipment | `equipment` | .150.11 | .150.12 | .150.13 | 150 | Ubuntu |

All addresses are `10.8.x.x`. Hostnames follow `cf-mes-<role>-0N` (`cf-eap-0N` for equipment). Hosts also belong to an OS group (`rhel` or `ubuntu`).

#### Hypervisors

The Proxmox hosts live in a **separate** inventory, `inventory/proxmox.yml`, so that playbooks targeting `hosts: all` (update, NTP, hardening, seal) can never touch them. Their connection settings are in `inventory/group_vars/proxmox.yml`:

```yaml
ansible_user: root
ansible_become: false
```

This file is needed because `group_vars/all.yml` outranks variables written inside an inventory file; without it, Ansible connects as `cloudfactoryadmin` and tries `sudo`, which Proxmox doesn't ship.

Install your key once:

```bash
for h in 10.8.10.101 10.8.10.102 10.8.10.103; do ssh-copy-id -i ~/.ssh/cf_homelab.pub root@$h; done
```

**Boot behaviour**, set per node:

| Setting | HYP-01 | HYP-02 | HYP-03 |
|---|---|---|---|
| Start-on-boot delay (System → Options) | 0 | 20s | 30s |

Per VM on every node: order `db → cache → cp → wrk → eap`, `onboot 1`. Proxmox shuts down in reverse order. The node delays favour HYP-01's database winning the leader election after a full cold start; they don't guarantee it.

***

### Playbooks

Run all commands from `CloudFactory/infra/Ansible`.

#### Bulk snapshot

Stops every MES VM, snapshots all fifteen under one name, and starts them again. All three hypervisors run in parallel, so the fleet is stopped at the same moment and the snapshots are consistent with each other.

```bash
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_snapshot.yml \
  -e snapshot=pre-etcd -e "description='Before forming the etcd cluster'"
```

- Names must start with a letter; letters, digits, `-` and `_` only; 40 characters max
- Refuses to force-stop: a VM that doesn't shut down within three minutes fails the run and nothing is snapshotted
- Won't overwrite an existing snapshot of the same name
- Snapshots grow on LVM-thin as VMs diverge — prune old ones

Clustered VMs must be snapshotted and rolled back **together**. Rolling back one etcd member while the others moved on leaves it with stale raft state.

#### Bulk shutdown

Shuts down HYP-02 and HYP-03 together, waits until both stop answering ping, pauses 10 seconds, shuts down HYP-01, then the FortiGate.

```bash
# VMs only
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_shutdown.yml -e confirm_shutdown=yes -e vms_only=true

# Everything, including the firewall
uv run ansible-playbook -i inventory/proxmox.yml playbooks/bulk_shutdown.yml -e confirm_shutdown=yes --ask-vault-pass
```

- `vms_only=true` runs `pvenode stopall` on each node and leaves the hypervisors and firewall up — the everyday option
- The full run needs physical access afterwards: the hypervisors and the FortiGate 61F must be powered back on by hand
- The vault is only opened in the FortiGate play; forgetting `--ask-vault-pass` means the hypervisors go down but the firewall stays up
- Once Patroni is running, `patronictl pause` before a planned full shutdown and `resume` afterwards, so no failover triggers while nodes go down

**FortiGate API access** — a REST API admin (`ansible-api`) with a profile allowing only System → Maintenance, Trusted Hosts limited to the control node. The token is in `vars/fortigate.vault.yml`:

```bash
export EDITOR=nano
uv run ansible-vault edit vars/fortigate.vault.yml
```

```yaml
fortigate_host: 10.8.10.1
fortigate_api_token: <token>
```

Test the token without shutting anything down:

```bash
uv run ansible localhost -e @vars/fortigate.vault.yml --ask-vault-pass -e ansible_become=false \
  -m uri -a 'url=https://{{ fortigate_host }}/api/v2/monitor/system/status headers={"Authorization":"Bearer {{ fortigate_api_token }}"} validate_certs=false'
```

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

#### Kubernetes node software

Installs CRI-O and the Kubernetes packages on `k8s_cluster`. Does **not** run `kubeadm init` or `join` — those are post-clone steps.

```bash
uv run ansible-playbook playbooks/k8s_node.yml
uv run ansible-playbook playbooks/k8s_node.yml --limit cf-mes-cp-01
```

Verify:

```bash
uv run ansible k8s_cluster -m shell -a "kubeadm version -o short; crio --version | head -1" -b
uv run ansible k8s_cluster -m shell -a "crictl --runtime-endpoint unix:///var/run/crio/crio.sock images" -b
```

**Requires the QEMU Guest Agent enabled on the VM** (Proxmox → Options → QEMU Guest Agent). The VM needs a full stop/start afterwards — a reboot does not add the virtio device.

#### PostgreSQL, Patroni and etcd

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

Patroni does not fail back automatically. After a failover, return leadership to db-01 with `patronictl switchover --candidate cf-mes-db-01`.

#### Forming the Patroni cluster

Post-clone step. Writes `/etc/etcd/etcd.conf` and `/etc/patroni/patroni.yml` from templates, opens the cluster ports, starts etcd on all three members together, then starts Patroni on db-01 (which runs `initdb`) before db-02 and db-03 clone from it.

Create the credentials once — the same vault password as the FortiGate file keeps it to one prompt:

```bash
cp vars/patroni.example.yml vars/patroni.vault.yml
nano vars/patroni.vault.yml
uv run ansible-vault encrypt vars/patroni.vault.yml
git status --short -uall vars/   # must list ONLY patroni.example.yml
```

The file holds four variables. **Single-quote every value** — an unquoted password starting with `!`, `&`, `*`, `@` or `%` is parsed by YAML as a tag or anchor and silently becomes empty:

```yaml
---
patroni_superuser_password: 'your-password'
patroni_replication_password: 'your-password'
patroni_rewind_password: 'your-password'
patroni_restapi_password: 'your-password'
```

The role refuses to run if any value is missing or shorter than `postgres_patroni_cluster_min_password_length` (default 8, in the role's `defaults/main.yml`). `openssl rand -base64 32` generates a strong value.

Check what Ansible actually reads, without printing the passwords — expect `str` and the real length for all four:

```bash
uv run ansible localhost -e @vars/patroni.vault.yml --ask-vault-pass -e ansible_become=false -m debug -a '{"msg": "{{ [patroni_superuser_password, patroni_replication_password, patroni_rewind_password, patroni_restapi_password] | map(\"type_debug\") | list }} {{ [patroni_superuser_password, patroni_replication_password, patroni_rewind_password, patroni_restapi_password] | map(\"string\") | map(\"length\") | list }}"}'
```

Then:

```bash
uv run ansible-playbook playbooks/postgres_patroni_cluster.yml --ask-vault-pass
```

- All three `patroni` hosts must be up — etcd cannot reach quorum with fewer, and the run fails at the health check
- "Wait for a healthy etcd cluster" and "Wait for every member to be running" retrying a few times is normal — replicas clone the leader with `pg_basebackup` first
- Patroni members are named after the inventory hostname (`cf-mes-db-01`…)
- `bootstrap.dcs` settings (PostgreSQL parameters) only apply at first bootstrap. Change them afterwards with `patronictl edit-config`, not by re-running the playbook
- Re-running is safe: configs are re-templated and Patroni is reloaded, running services are left alone

Verify from the control node:

```bash
uv run ansible cf-mes-db-01 -b -m shell -a "sudo -u postgres patronictl -c /etc/patroni/patroni.yml list"
```

Expect one `Leader` and two `Replica` members, all `streaming`, lag 0:

```
| Member       | Host        | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
| cf-mes-db-01 | 10.8.103.11 | Leader  | running   |  1 |             |     |            |     |
| cf-mes-db-02 | 10.8.103.12 | Replica | streaming |  1 |   0/5000060 |   0 |  0/5000060 |   0 |
| cf-mes-db-03 | 10.8.103.13 | Replica | streaming |  1 |   0/5000060 |   0 |  0/5000060 |   0 |
```

On a database node itself, the config is `0600 postgres`, so run `patronictl` as postgres: `sudo -u postgres patronictl -c /etc/patroni/patroni.yml list`.

Once the cluster is formed, the three database VMs are clustered: snapshot and roll them back **together**.

#### Resetting Patroni state

Stops Patroni and etcd, deletes the etcd member data and the PostgreSQL data directory. **Destroys the database.** Needed when a host carries state from a previous cluster — for example a hand-built node, or a member whose etcd raft state diverged.

```bash
uv run ansible-playbook playbooks/postgres_patroni_reset.yml -e confirm_reset=yes --limit cf-mes-db-01
```

Refuses to run without `confirm_reset=yes`. Reset every member together before re-forming — a lone reset member cannot rejoin an etcd cluster that still lists it with old state.

#### HAProxy and the database virtual IP

Gives applications one address that always reaches the current Patroni leader. HAProxy runs on every `patroni` host and asks each node's Patroni API (`:8008/primary`, `/replica`) which role it holds; keepalived floats a virtual IP between the three HAProxy instances.

| Endpoint | Routes to |
|---|---|
| `10.8.103.10:5000` | current leader — read/write |
| `10.8.103.10:5001` | replicas, round robin — read-only |
| `http://10.8.103.10:7000` | HAProxy status page |

```bash
uv run ansible-playbook playbooks/postgres_patroni_haproxy.yml
```

- Run after `postgres_patroni_cluster.yml` — HAProxy needs Patroni answering on `:8008`
- The VIP (`postgres_patroni_haproxy_vip`, default `10.8.103.10`) must be free and outside any DHCP range. The first run pings it and refuses if something already answers
- keepalived uses unicast VRRP between the three nodes, router ID `103` (`postgres_patroni_haproxy_vrrp_id`) — change it if anything else on VLAN 103 runs VRRP
- db-01 has the highest priority, so the VIP normally sits there. It moves only if HAProxy or the VM on the holder dies — **not** on a Patroni failover, which HAProxy handles by itself
- Sets the SELinux boolean `haproxy_connect_any`; RHEL's policy otherwise blocks HAProxy from ports 5000, 5001, 7000, 5432 and 8008
- The run ends by asserting that exactly one node holds the VIP

Verify:

```bash
uv run ansible patroni -b -m shell -a "ip -4 -br addr show ens18; systemctl is-active haproxy keepalived"
psql -h 10.8.103.10 -p 5000 -U postgres -c "select inet_server_addr(), pg_is_in_recovery();"   # leader's IP, f
psql -h 10.8.103.10 -p 5001 -U postgres -c "select inet_server_addr(), pg_is_in_recovery();"   # a replica's IP, t
```

Connections through HAProxy reach PostgreSQL from the HAProxy node's address, so `pg_hba.conf` sees VLAN 103 rather than the client. Access control for application subnets therefore lives in the FortiGate policy (e.g. VLAN 101 → `10.8.103.10:5000`), not in `pg_hba`.

Failover test:

```bash
uv run ansible cf-mes-db-01 -b -m shell -a "sudo -u postgres patronictl -c /etc/patroni/patroni.yml switchover --candidate cf-mes-db-02 --force"
# :5000 now reaches db-02 without the VIP moving; switch back:
uv run ansible cf-mes-db-01 -b -m shell -a "sudo -u postgres patronictl -c /etc/patroni/patroni.yml switchover --candidate cf-mes-db-01 --force"
```

#### TimescaleDB

Adds the TimescaleDB extension to the running cluster for equipment time-series data (event reports, trace data, status variables). **No `patronictl pause` needed.**

```bash
uv run ansible-playbook playbooks/postgres_patroni_timescaledb.yml --ask-vault-pass
```

What it does, in order:

1. Installs `timescaledb-2-postgresql-18` from Timescale's own repository on **all three** members — replicas replay the leader's WAL and can become leader, so they need the binaries too. Stops before changing anything if Timescale publishes no matching build for this OS.
2. PATCHes the Patroni cluster configuration (REST API, vaulted password): `shared_preload_libraries = timescaledb`, telemetry off, `timescaledb.max_background_workers = 8`, `max_worker_processes = 19`
3. Rolling restart, one member at a time — replicas first, leader last. Patroni restarts PostgreSQL in place: a few seconds of write downtime while the leader restarts, **no failover**
4. Asserts every member preloaded TimescaleDB, then runs `CREATE EXTENSION` on the leader in the listed databases (none by default)

Once a database exists, create the extension in it:

```bash
uv run ansible-playbook playbooks/postgres_patroni_timescaledb.yml --ask-vault-pass \
  -e '{"postgres_patroni_timescaledb_databases": ["mes"]}'
```

- Re-running is safe: unchanged parameters are not PATCHed and members without a pending restart are skipped
- **Pin the version** after the first run: the final task prints it; set `postgres_patroni_timescaledb_version` in the role defaults (e.g. `2.23.1-0.el10`)
- Run it **after** `postgres_patroni_cluster.yml` on a rebuild — a fresh bootstrap does not preload TimescaleDB
- Upgrading the package later: install the new version on all members, rolling-restart, then on the leader in each database, in a fresh session: `psql -X -d mes -c "ALTER EXTENSION timescaledb UPDATE;"`
- To preload more libraries later, list them together in `postgres_patroni_timescaledb_parameters`: `timescaledb,pg_stat_statements`

Verify:

```bash
uv run ansible patroni -b -m shell -a "runuser -u postgres -- psql -XAtc 'show shared_preload_libraries'"
```

#### Redis and Sentinel

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

***

### Sealing a VM for cloning

Strips host identity so clones don't collide. **Destructive** — run only against the VM you intend to template.

Before sealing:

1. Apply all pending updates and reboot
2. Detach the CD/DVD drive — restores on other nodes fail if the ISO isn't present there
3. Shut down the VM cleanly
4. Take a Proxmox snapshot (`pre-seal`) as a rollback point
5. Boot it back up

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

After it completes, shut down and **do not boot the VM again** — booting regenerates machine-id and SSH host keys, undoing the seal. Back it up with `vzdump` (Mode: Stop) for transfer to another node. Empty `template_source` when you're not actively sealing.

#### What sealing clears

| Item | Reason |
|---|---|
| `/etc/machine-id`, `/var/lib/dbus/machine-id` | DHCP identifier — clones would request the same lease |
| SSH host keys | clones would present identical fingerprints |
| cloud-init state | without a clean, cloud-init won't re-run and the clone keeps the source hostname |
| `/etc/hostname` | set per-clone |
| logs, package caches, shell history | dead weight in every clone |

It also asserts that no cluster state exists (`/etc/kubernetes/pki`, `/var/lib/etcd/member`, `/var/lib/redis/dump.rdb`) and fails if it finds any.

#### Personalising a clone

Restore with **unique** ticked (new MAC), enable the QEMU Guest Agent, detach any CD drive, then from the Proxmox console:

```bash
# RHEL
sudo nmcli con mod ens18 ipv4.method manual ipv4.addresses <ip>/24 ipv4.gateway <vlan>.1 ipv4.dns <vlan>.1
sudo nmcli con up ens18
sudo hostnamectl set-hostname <name>
sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1 <name>/' /etc/hosts
sudo subscription-manager clean
sudo subscription-manager register
sudo systemctl enable --now qemu-guest-agent   # + crio on cp and wrk

# Ubuntu — sshd won't start until host keys are regenerated
sudo ssh-keygen -A && sudo systemctl restart ssh
# then the same nmcli / hostnamectl steps, using connection netplan-ens18,
# and retire stale netplan files:
sudo mv /etc/netplan/01-netcfg.yaml /etc/netplan/00-installer-config.yaml /root/
```

Then from the control node: `ssh-keyscan -H <ip> >> ~/.ssh/known_hosts`, and re-run the fleet identity check.

#### Red Hat subscriptions

Sealing does **not** unregister from Red Hat. The clone boots carrying the source's consumer certificate, so each clone must drop it before registering itself — `clean` first, then `register`. Registering on top of inherited certificates either fails or creates a duplicate entry. Register **after** setting the hostname, or the portal records the wrong name.

Because sources are never unregistered, their entries remain in the Red Hat portal. Prune them periodically — a Developer Subscription allows 16 systems.

***

### Lessons learned

**Stale netplan files on Ubuntu clones.** The golden image carried a hand-written `01-netcfg.yaml` with a static address. After a clone's IP was changed with `nmcli`, netplan *merged* both files and the VM came up with **two** addresses — the old one silently colliding with another VM. It went unnoticed for a week, and during a readdress it caused commands aimed at one VM to land on another. Always retire `01-netcfg.yaml` and `00-installer-config.yaml` on Ubuntu clones, and check `ip -4 -br addr` shows exactly one address.

**Target by MAC, not IP, when addresses might overlap.** When two VMs share an address, whichever answers ARP first gets your command. Use `qm guest exec <vmid>` from the hypervisor instead — it reaches one specific VM with no network involved. RHEL's guest agent blocks `guest-exec` by default; Ubuntu's allows it.

**Renaming a Proxmox node.** Back up `/etc/pve` and **check the backup exists**. VM configs live under `/etc/pve/nodes/<name>/qemu-server/`; move them to the new node directory, confirm with `qm list`, and only then remove the old directory. VM IDs must be unique across `/etc/pve`, so this has to be `mv`, not `cp`. Regenerate certificates with `pvecm updatecerts --force` after both the rename and any IP change.

**One-node clusters.** HYP-03 had a leftover one-node cluster (`CF-Cluster-01`) in `/etc/corosync/`. After the rename, corosync couldn't find itself in its own member list, pmxcfs lost quorum, and `/etc/pve` hung — breaking the web UI, `updatecerts` and `rm`. Check `/etc/corosync/` before renaming or re-addressing a node; convert to standalone with Proxmox's "separate a node without reinstalling" procedure.

**`/etc/hosts` on hypervisors** must resolve the node's hostname to its real address — `hostname -i` must return the management IP, not `127.0.1.1` or an old address.

**`ansible.cfg` duplicate keys.** Ansible refuses a config with a key repeated in the same section, and nothing Ansible-related runs until it's fixed.

**Patroni gave up at boot.** After a cold start of all three DB VMs, Patroni on one node started before etcd had quorum, logged `EtcdConnectionFailed: No more machines in the cluster` and exited — and the PGDG unit has no restart policy, so it stayed down. The cluster role now installs `/etc/systemd/system/patroni.service.d/10-etcd-restart.conf` (`After=etcd.service`, `Restart=on-failure`).

**Pause survives reboots.** `patronictl pause` is stored in etcd, not in the process, so it is still on after the VMs restart — and while paused, Patroni does not start PostgreSQL on a rebooted node (it shows `stopped`). `patronictl list` prints `Maintenance mode: on` as the tell. Always `resume` after a planned shutdown.

**Unquoted secrets in YAML.** A vault value such as `patroni_superuser_password: !Secret123` loads as *empty*: YAML reads the leading `!` as a type tag (`&` and `*` as anchors/aliases). The raw file looks correct, `ansible-vault view` shows 11 characters, but Ansible sees `NoneType`. Single-quote every secret, and check with `type_debug` rather than trusting the file's appearance.

**Variable precedence.** `ansible_*` connection variables beat task keywords. `become: false` on a task loses to `ansible_become: true` from `group_vars`; use `vars: { ansible_become: false }` on the task instead.

***

### Pinned versions

| Component | Version | Source |
|---|---|---|
| Kubernetes | 1.37.0 | `pkgs.k8s.io/core:/stable:/v1.37` |
| CRI-O | 1.36.5 | `download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.36` |
| PostgreSQL | 18.6-4PGDG.rhel10.2 | PGDG |
| Patroni | 4.1.5-1PGDG.rhel10.2 | PGDG (`patroni-etcd`) |
| TimescaleDB | *pin after first install* | `packagecloud.io/timescale/timescaledb` |
| etcd | 3.7.1 | GitHub release tarball |
| Redis / Sentinel | 6:8.10.1-1rl1~resolute1 | `packages.redis.io/deb` |
| RHEL | 10.2 | Red Hat CDN |
| Ubuntu | resolute | Ubuntu archive |
| Proxmox VE | 9.2.2 | Proxmox |
| FortiOS | 7.2.13 (FortiGate 61F) | Fortinet |

CRI-O intentionally trails Kubernetes by one minor — this is the upstream-documented pairing, not a workaround.

Kubernetes and CRI-O are pinned by `exclude` in their repo definitions; install tasks pass `disable_excludes` to bypass deliberately. PostgreSQL, Patroni and Redis are pinned by explicit version string, with Redis additionally held via `dpkg_selections`.

Patroni's etcd cluster is **separate from the Kubernetes control plane's etcd**. Do not point one at the other — a cluster rebuild would take the database's coordination layer with it.

***

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

Files must end with a single newline and have no trailing spaces — `pre-commit install` in the repo root fixes both automatically on commit.