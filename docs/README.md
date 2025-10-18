## Samba Service

Declarative Samba 4.21 fileserver definition powered by the shared infrastructure adapters. The role renders host-independent manifests for Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or a bare-metal systemd unit while keeping configuration and credentials portable.

### Runtime Coverage
- Proxmox LXC (packages + config injected via `service_files`)
- Docker Compose v2 (config delivered as Docker secret)
- Podman Quadlet
- Kubernetes Deployment + Service + Secret + PVC
- Bare-metal systemd using the bundled configuration and tmpfs mounts

### Exports
```
SAMBA_SERVER={{ service_ip }}
SAMBA_SHARE={{ samba_primary_share_name }}
SAMBA_SHARE_URL={{ samba_primary_share_url }}
SAMBA_SHARE_PATH={{ samba_primary_share_path }}
```

The role writes these values to `exports/{{ service_id }}.env` (override with
`service_exports_env_file`) so downstream automation can source the canonical
share coordinates during provisioning.

### Secrets
- `samba-config` → rendered YAML config mounted via Docker/Podman secrets
- `samba-admin-password` → stored separately and referenced from the config

### Mounts
- Persistent: `/data` (YAML config, state), `/shares` (share contents)
- Ephemeral: `/run/samba` tmpfs across container and bare-metal targets

### Health Check
Runs `smbstatus` inside the container / service to ensure `smbd` is online. The same probe feeds Compose healthchecks, Quadlet checks, Kubernetes readiness/liveness, and the post-deploy validation gate.

### Edge Ingress
SMB is not exposed over an HTTP edge; `edge_ingress` is intentionally not
implemented for this service.

### Key Overrides
| Variable | Default | Purpose |
| --- | --- | --- |
| `samba_service_port_smb` | `445` | SMB TCP port |
| `samba_service_port_netbios` | `139` | Legacy NetBIOS port (optional) |
| `samba_primary_share_name` | `share` | Default exported share name |
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
        samba_admin_password: "{{ vault_samba_admin_password }}"
        samba_share_volume_size_gb: 500
```

