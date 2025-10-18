## Samba Service

Declarative Samba 4.21 fileserver definition powered by the shared infrastructure adapters. The role renders host-independent manifests for Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or a bare-metal systemd unit while keeping configuration and credentials portable. Authentication, authorization, data-protection, and security hardening are first-class citizens for production environments.

### Runtime Coverage
- Proxmox LXC (packages + config injected via `service_files`)
- Docker Compose v2 (config delivered as Docker secret)
- Podman Quadlet
- Kubernetes Deployment + Service + Secret + PVC
- Bare-metal systemd using the bundled configuration and tmpfs mounts

### Exports
```
SAMBA_SERVER={{ service_ip }}
SAMBA_PROTOCOL={{ samba_protocol_version }}
SAMBA_SHARE={{ samba_primary_share_name }}
SAMBA_SHARE_URL={{ samba_primary_share_url }}
SAMBA_SHARE_SMB_URL={{ samba_primary_share_smb_url }}
SAMBA_SHARE_PATH={{ samba_primary_share_path }}
SAMBA_SHARE_MOUNT_PATH={{ samba_share_mount_path }}
SAMBA_EXTERNAL_HOSTNAME={{ samba_external_hostname }}
```

The role writes these values to `exports/{{ service_id }}.env` (override with
`service_exports_env_file`) so downstream automation can source the canonical
share coordinates during provisioning. SMB URLs and DNS records help clients
discover the service without hard-coding IP addresses.

### Secrets
- `samba-config` → rendered YAML config mounted via Docker/Podman secrets
- `samba-password-<username>` → per-user password files, never exposed via
  container environment variables

### Mounts
- Persistent: `/data` (YAML config, state), `/shares` (share contents)
- Ephemeral tmpfs mounts: `/run/samba`, `/var/log/samba`, `/var/cache/samba`
- Bind: `/etc/localtime` so timezone propagation works across runtimes

### Health Check
Runs `smbstatus` inside the container / service to ensure `smbd` is online. If
`samba_wsdd2_enable` is true the probe additionally verifies that `wsdd2`
responds, preventing a green deployment when network discovery is offline.

### Edge Ingress
SMB is not exposed over an HTTP edge; `edge_ingress` is intentionally not
implemented for this service.

### Authentication & Authorization

- `samba_users` is a list of `{name, uid, gid, group, password, password_file}`
  dictionaries. Every entry is materialized as its own secret
  (`samba-password-<name>`) and referenced from the generated config file, so
  credentials never leak into runtime environment variables.
- Optional Active Directory support is exposed via `samba_ad_domain` and
  `samba_ad_server`. When set, the role enables ADS security, sets the Kerberos
  realm, and publishes the domain in `SAMBA_REALM`.
- Shares are defined by `samba_shares`. Each share supports `guest_ok`,
  `browsable`, `read_only`, `valid_users`, `read_list`, `write_list`,
  `force_user`, `force_group`, `veto_files`, and `extra_options`. This enables
  mixed read/write permissions, hidden administrative shares, and public guests
  without affecting the rest of the configuration.
- `write_list` and `read_list` parameters let you express read-only consumers
  while still granting write access to privileged accounts.

### Shares & Filesystems

- Multiple shares can be defined with arbitrary absolute paths, enabling folder
  hierarchies such as `/shares/departments/hr` or `/srv/archive/2024`.
- `samba_force_user` / `samba_force_group` (defaulting to the admin account)
  guarantee consistent on-disk ownership even though the container runs as an
  unprivileged UID/GID 1000.
- Per-share recycle bins can be enabled with `recycle_bin: true`, exposing
  `recycle_*` tuning knobs like `samba_recycle_bin_expire` to prevent unlimited
  growth.
- Optional `samba_quota_enabled`, `samba_full_audit_enabled`, and
  `samba_shadow_copy_enabled` switches enable the corresponding Samba VFS
  modules for quota enforcement, audit logging, and snapshot surfacing (Previous
  Versions). Combine these with filesystem-level tooling (ZFS/Btrfs/LVM) for
  robust data protection.

### Networking & Protocols

- SMB3 is the default dialect (`samba_protocol_version`), SMB1 is disabled, and
  server signing is mandatory by default to block downgrade and MITM attacks.
- Bind-to-interface defaults prevent accidental IPv6 exposure. Override
  `samba_interfaces` if you need to expose additional NICs or VLANs.
- Published ports now include TCP/445, TCP/139, UDP/137, and UDP/138 so that
  NetBIOS name- and datagram-services remain reachable when legacy clients need
  them.

### Security Hardening

- Containers run as UID/GID 1000 with `read_only_root_filesystem: true`, tmpfs
  mounts for mutable paths, and capabilities reduced to `CAP_NET_BIND_SERVICE`.
- AppArmor/SELinux integration is exposed via `samba_apparmor_profile` and
  `samba_selinux_context`, constraining file access to the share hierarchy.
- The default LXC profile no longer requests nested container support unless it
  is explicitly required.

### Operational Guidance

- `samba_workgroup` is validated against reserved names and must be uppercase
  A–Z/0–9/underscore (max 15 characters) for interoperability.
- Backups are still operator-driven. Mount `/shares` into your preferred backup
  tool (e.g., Restic) or enable `samba_shadow_copy_enabled` plus filesystem
  snapshots to provide Previous Versions. Documented exports and deterministic
  secrets make integrating with external backup orchestration straightforward.
- Stateful clustering (CTDB) is not configured. Run a single replica or bring
  your own CTDB stack if you must scale horizontally—simultaneous writers over a
  shared NFS backend risk lock corruption.
- A migration checklist is recommended when moving from an existing SMB server:
  1. Copy data with `rsync -a` (preserves ACLs/attributes).
  2. Validate share definitions (`samba_shares`) mirror the source exports.
  3. Import users via `samba_users` and rotate passwords or Kerberos keytabs.
  4. Schedule downtime to swap DNS to `samba_external_hostname` and test with
     `smbstatus`/`smbclient` before reopening access.
- Remote/edge access is intentionally out-of-scope. Pair the service with SMB-
  over-QUIC, a VPN, or HTTPS tunneling if you must expose it beyond the LAN.

### Key Overrides
| Variable | Default | Purpose |
| --- | --- | --- |
| `samba_service_port_smb` | `445` | SMB TCP port |
| `samba_service_port_netbios` | `139` | NetBIOS session service |
| `samba_service_port_nmb` | `137` | NetBIOS name service (UDP) |
| `samba_service_port_dgram` | `138` | NetBIOS datagram service (UDP) |
| `samba_users` | `[{name,uid,gid,password,…}]` | List of Samba accounts |
| `samba_shares` | `[{name,path,...}]` | Declarative share catalog |
| `samba_protocol_version` | `SMB3` | Advertised SMB dialect |
| `samba_full_audit_enabled` | `false` | Enable Samba VFS audit logging |
| `samba_shadow_copy_enabled` | `false` | Surface filesystem snapshots |
| `samba_share_volume_size_gb` | `200` | Persistent storage allocation for shares |
| `samba_config_volume_size_gb` | `2` | Persistent storage for config/state |
| `samba_container_vmid` | `240` | Proxmox VMID |
| `samba_container_memory_mb` | `3072` | Memory reservation |
| `samba_container_cpu_cores` | `2` | vCPU allocation |
| `samba_kubernetes_namespace` | `files` | Namespace for the workload |

### Usage
```yaml
- hosts: fileserver_hosts
  roles:
    - role: svc-samba
      vars:
        runtime: podman
        samba_users:
          - name: filesvc
            uid: 10050
            gid: 10050
            password: "{{ vault_samba_filesvc_password }}"
        samba_shares:
          - name: public
            path: /shares/public
            guest_ok: true
            browsable: true
            read_only: false
            write_list: "filesvc"
          - name: finance
            path: /shares/departments/finance
            read_only: false
            valid_users: "filesvc,controller"
            read_list: "auditor"
            veto_files:
              - "*.exe"
              - "Thumbs.db"
            recycle_bin: true
        samba_full_audit_enabled: true
        samba_shadow_copy_enabled: true
        samba_external_hostname: fileserver.internal.example.com
```

See `defaults/main.yml` for the authoritative list of tunables and security
defaults.
