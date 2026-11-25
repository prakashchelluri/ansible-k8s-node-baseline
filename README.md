# ansible-k8s-node-baseline

Ansible roles that give RHEL/Rocky 8-9 and Ubuntu 22.04/24.04 nodes a consistent, Kubernetes-ready baseline.

> Lab project: a clean-room reference implementation of a pattern I use at work. It contains no employer code or data.

## The problem

Clusters inherit whatever their nodes were built with. When nodes come from different images, teams or years, you get
clock drift that breaks certificate validation and log correlation, journals that fill `/var`, swap left on so the kubelet
refuses to start, `br_netfilter` missing after a reboot so Service traffic silently bypasses iptables, no node metrics,
and SSH that still accepts passwords. Each of these is a small, well-known fix; the hard part is applying all of them
the same way on every node, every time, and being able to prove it with `--check`.

These roles encode that baseline once, for both the EL and Ubuntu families.

## Quick start

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r collections/requirements.yml -p collections

# Edit inventory/example/hosts.yml and group_vars to point at your nodes, then preview:
ansible-playbook site.yml --check --diff

# Apply:
ansible-playbook site.yml

# Only one role:
ansible-playbook site.yml --tags node_exporter
```

The `hardening_basic` role turns off SSH password authentication. Make sure your public key is on the hosts first.

## How it works

`site.yml` applies four roles, in order, to the `k8s_nodes` group:

| Role | What it does |
|------|--------------|
| `baseline` | Installs common packages, configures chrony, caps journald disk usage, adds logrotate policies, writes sysctl and process limits. |
| `k8s_prereqs` | Turns swap off now and in `/etc/fstab`, loads `overlay` and `br_netfilter` now and on boot, sets the bridge and IP-forwarding sysctls, installs conntrack/socat/ethtool/iptables. |
| `node_exporter` | Downloads a pinned node_exporter release, verifies its SHA-256, installs it under `/opt/node_exporter/<release>` with a symlink in `/usr/local/bin`, and runs it as an unprivileged user under a hardened systemd unit with a textfile collector directory. |
| `hardening_basic` | Installs an sshd drop-in (no root login, no passwords, key-only) and enables unattended security updates: `unattended-upgrades` on Ubuntu, `dnf-automatic` on EL. |

Design choices:

- Modules only; the two `command` tasks (`swapoff -a` and an sshd reload) have no module equivalent and are guarded.
- Everything is idempotent: a second run reports `changed=0` (CI enforces this).
- OS differences live in each role's `vars/RedHat.yml` and `vars/Debian.yml`, not in `when:` sprawl.
- Config is written to drop-in files (`/etc/sysctl.d/`, `/etc/systemd/journald.conf.d/`, `/etc/ssh/sshd_config.d/`,
  `/etc/security/limits.d/`) so package updates never overwrite it.
- sshd uses the first value it reads, so the drop-in is named `00-hardening.conf` to win over cloud-image drop-ins
  that re-enable passwords. On EL 8, whose stock `sshd_config` has no `Include`, the role adds it.

### CI

`.github/workflows/ci.yml` runs on every push and pull request:

1. `yamllint --strict`, `ansible-lint` (production profile) and `ansible-playbook --syntax-check`.
2. Converges the GitHub Actions runner itself (Ubuntu, real systemd and kernel) with `tests/localhost.yml`, runs the
   playbook a second time and fails if anything changed, then checks the result: node_exporter serves metrics,
   chrony is active, `ip_forward` and `bridge-nf-call-iptables` are 1, swap is off, `br_netfilter` is loaded and
   `sshd -T` shows root login and passwords disabled.

There is no Molecule scenario. The Docker driver cannot load kernel modules, turn swap off or set non-namespaced sysctls
inside a container, so it would only test the roles with their most important tasks disabled; converging a real VM
covers more.

## Variables

Every variable has a default in `roles/<role>/defaults/main.yml`. Override them in `group_vars` or `host_vars`
(see `inventory/example/group_vars/k8s_nodes.yml`).

### baseline

| Variable | Default | Description |
|----------|---------|-------------|
| `baseline_packages` | chrony, curl, jq, logrotate, lsof, rsync, tar | Packages on every node. |
| `baseline_packages_extra` | `[]` | Additional packages. |
| `baseline_chrony_pools` | `[pool.ntp.org]` | chrony `pool` entries. |
| `baseline_chrony_servers` | `[]` | chrony `server` entries (e.g. internal NTP). |
| `baseline_journald_storage` | `persistent` | journald `Storage=`. |
| `baseline_journald_system_max_use` | `512M` | journald `SystemMaxUse=`. |
| `baseline_journald_system_max_file_size` | `64M` | journald `SystemMaxFileSize=`. |
| `baseline_journald_max_retention` | `2week` | journald `MaxRetentionSec=`. |
| `baseline_logrotate_configs` | `[]` | List of `{name, paths, frequency, rotate}`; one file per entry in `/etc/logrotate.d/`. |
| `baseline_sysctl` | see defaults | Dict written to `/etc/sysctl.d/60-baseline.conf` (inotify, file-max, somaxconn, max_map_count, swappiness). |
| `baseline_limits` | nofile/nproc 65536 | List of `{domain, type, item, value}` for `/etc/security/limits.d/90-baseline.conf`. |

### k8s_prereqs

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_prereqs_disable_swap` | `true` | Turn swap off now and comment out swap lines in `/etc/fstab`. |
| `k8s_prereqs_kernel_modules` | `[overlay, br_netfilter]` | Loaded now and via `/etc/modules-load.d/k8s.conf`. |
| `k8s_prereqs_sysctl` | bridge-nf-call-iptables/ip6tables=1, ip_forward=1 | Dict written to `/etc/sysctl.d/99-kubernetes.conf`. |
| `k8s_prereqs_packages_extra` | `[]` | Added to the OS-specific runtime prerequisites. |

### node_exporter

| Variable | Default | Description |
|----------|---------|-------------|
| `node_exporter_version` | `1.13.1` | Pinned release. |
| `node_exporter_checksums` | amd64 and arm64 SHA-256 | Must match the release's `sha256sums.txt`; update together with the version. |
| `node_exporter_download_base_url` | GitHub releases | Point at an internal mirror for air-gapped sites. |
| `node_exporter_user` | `node-exporter` | System user the service runs as. |
| `node_exporter_install_dir` | `/opt/node_exporter` | Where release tarballs are unpacked. |
| `node_exporter_binary_path` | `/usr/local/bin/node_exporter` | Symlink to the active release. |
| `node_exporter_listen_address` | `0.0.0.0:9100` | `--web.listen-address`. |
| `node_exporter_textfile_dir` | `/var/lib/node_exporter/textfile_collector` | Drop `*.prom` files here for custom metrics. |
| `node_exporter_extra_args` | `[]` | Extra flags, e.g. `--collector.systemd`. |

### hardening_basic

| Variable | Default | Description |
|----------|---------|-------------|
| `hardening_basic_ssh_permit_root_login` | `"no"` | sshd `PermitRootLogin`. |
| `hardening_basic_ssh_password_authentication` | `"no"` | sshd `PasswordAuthentication`. |
| `hardening_basic_ssh_kbd_interactive_authentication` | `"no"` | sshd `KbdInteractiveAuthentication`. |
| `hardening_basic_ssh_x11_forwarding` | `"no"` | sshd `X11Forwarding`. |
| `hardening_basic_ssh_max_auth_tries` | `4` | sshd `MaxAuthTries`. |
| `hardening_basic_auto_updates` | `true` | Enable unattended security updates. |
| `hardening_basic_dnf_upgrade_type` | `security` | EL only: `security` or `default` (all updates). |

## Limitations

- Only the Ubuntu path is exercised end to end in CI. The EL path passes lint and syntax checks but is not converged
  in CI; test it on a Rocky/RHEL VM before relying on it.
- `--check` is exact on a node that has been converged once. On a brand-new node some tasks depend on packages or
  downloads that check mode does not perform, so those tasks are allowed to fail in check mode only
  (`ignore_errors: "{{ ansible_check_mode }}"`) and the real run will differ slightly from the preview.
- It does not install a container runtime, kubeadm or kubelet, and it does not configure SELinux or a host firewall;
  those choices depend on the distribution of Kubernetes you run.
- node_exporter is installed from GitHub releases. Without internet access, set `node_exporter_download_base_url` to
  a mirror that keeps the same path layout.
- `swapoff -a` disables all swap, including zram; nodes that need swap for kubelet's `NodeSwap` feature should set
  `k8s_prereqs_disable_swap: false`.

## License

MIT, see [LICENSE](LICENSE).
