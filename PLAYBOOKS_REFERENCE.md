# Playbooks Reference — for Web UI Integration

This document describes every Ansible playbook in this repo that's meant to be triggered from AWX — what it does, what variables it accepts, and exactly what JSON to send as `extra_vars` when launching it via the AWX API. It's written for a developer building a web UI on top of AWX, not for someone reading the Ansible source directly.

There are **7 playbooks** in three groups:

| Group | Playbook | Purpose |
|---|---|---|
| Create | `Create_VM_From_MultiVM_Template.yml` | Create a VM on Proxmox, join it to the tailnet, optionally expose it publicly in the same run |
| Create | `Create_LXC_App.yml` | Create an LXC container directly on Proxmox, join it to the tailnet, optionally expose it publicly in the same run |
| Create | `Install_Docker_And_App.yml` | Install Docker on an existing VM and deploy one or more containers on it |
| Expose | `Expose_App_To_Internet.yml` | Make an already-running service on a VM reachable from the public internet, without redeploying anything else |
| Delete | `Delete_VM.yml` | Destroy a VM (and its underlying template, if you pass a template name) |
| Delete | `Delete_LXC.yml` | Destroy an LXC container |
| Delete | `Delete_Docker_App.yml` | Remove one or more Docker apps from a VM, leaving Traefik and other apps running |

All of them are idempotent by name — re-running a create playbook with the same name reuses/updates the existing resource instead of creating a duplicate; re-running a delete playbook against a name that doesn't exist is a harmless no-op.

**Design principle: simple by default, capable when asked.** Each Create playbook's variables below are split into **Simple** (what a basic UI form needs — a handful of fields, sensible defaults) and **Advanced** (optional power-user knobs, left blank/default for the common case). Anything the playbook can determine by asking the Proxmox host itself — target architecture (amd64/arm64), storage backend availability, network bridge — is validated against that host **every run**, not just when the field is left blank, and auto-corrected if it's wrong. This matters in practice: a stale AWX Survey default, an old saved `extra_vars` blob, or a value copied from a different host are all just as invalid as never setting the field at all, and the playbook treats them the same way — check the real host, trust it over whatever was passed in. Anything that would otherwise require the caller to write a shell command or know infrastructure trivia (an app's install steps, its default port) is instead picked from a built-in catalog by a plain name (Create LXC's `app_choice`, Install Docker's `docker_apps[].name`) — raw scripting fields like `app_install_script` exist underneath as an admin/operator escape hatch, not something a normal end user should ever see as a text box. Build the UI's default form from the Simple tables and catalog names only; put Advanced fields behind a collapsed "Advanced" section, not the main flow.

---

## 1. How to launch these from a web UI (AWX REST API)

Each playbook is wired up as its own AWX **Job Template**. To launch one:

```
POST https://<awx-host>/api/v2/job_templates/<template_id>/launch/
Authorization: Bearer <awx-api-token>
Content-Type: application/json

{
  "extra_vars": { ...variables for this playbook, see sections below... }
}
```

Notes:
- The Job Template must have **"Prompt on Launch" enabled for Variables** for this to work — otherwise anything you send in `extra_vars` is silently ignored and the template's own saved defaults are used instead. This is a real gotcha we hit during testing: confirm this checkbox is on for every template before wiring up the UI.
- The response includes a `job` ID (an integer). Poll job status at `GET /api/v2/jobs/<job_id>/` (look at the `status` field: `pending` → `running` → `successful`/`failed`).
- Get the full console output at `GET /api/v2/jobs/<job_id>/stdout/?format=txt` (or `format=json` for structured event data).
- Auth: create a Personal Access Token for a service-account user in AWX (**Users → your user → Tokens**) and use it as a Bearer token, or use HTTP Basic Auth with a service account's username/password. Bearer token is recommended.
- Get each Job Template's numeric ID from `GET /api/v2/job_templates/?name=<template-name>`.

### Required AWX Credentials per playbook

Two different kinds of SSH credential are used across these playbooks — don't mix them up:

- **Proxmox-host credential**: SSH access to the Proxmox host itself (e.g. `qa2`), used by any playbook that runs `qm`/`pct`/`pvesh` commands directly on Proxmox.
- **VM credential**: SSH access to a specific *created* VM (as the `pronto` user, key-based — the private key matching `id_rsa.pub` in this repo), used by any playbook that installs/manages things *inside* a VM over SSH.
- **Tailscale Auth Key credential**: a Custom Credential Type in AWX that injects `tailscale_authkey` as an extra_var. Backed by a **Reusable + Tagged** Tailscale auth key (not Ephemeral). Only needed by the two playbooks that join something to the tailnet for the first time.

| Playbook | Credential(s) needed |
|---|---|
| Create VM | Proxmox-host SSH + Tailscale Auth Key |
| Create LXC | Proxmox-host SSH + Tailscale Auth Key |
| Install Docker And App | VM SSH only |
| Expose App To Internet | VM SSH only |
| Delete VM | Proxmox-host SSH only |
| Delete LXC | Proxmox-host SSH only |
| Delete Docker App | VM SSH only |

---

## 2. Create VM — `Create_VM_From_MultiVM_Template.yml`

Ensures the requested OS template exists on Proxmox (builds it on first use, reuses it after), clones a VM from it, and joins the VM to the tailnet. Re-running with the same `vm_name` reuses the existing VM rather than cloning a duplicate.

**Connects to:** the Proxmox host directly (`qm`, `pvesh` commands).

### Simple — the UI's default form should only need these

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `os_type` | string (enum) | yes | `"ubuntu-24.04"` | One of: `ubuntu-24.04`, `ubuntu-22.04`, `debian-12` |
| `vm_name` | string | yes | `"dev-vm01"` | The VM's name and its tailnet hostname |
| `memory` | integer | no | `2048` | RAM in MB |
| `cores` | integer | no | `2` | CPU cores |
| `disk_size` | string | no | `"15G"` | Disk size, Proxmox `qm resize` format (a number + `G`) |
| `expose_to_internet` | boolean | no | `false` | If true, exposes `expose_port` on this VM publicly via Tailscale Funnel, once approved — one job creates **and** exposes it, no separate step needed |
| `expose_port` | integer | no | `80` | Local port to expose when `expose_to_internet` is true |
| `tailscale_authkey` | string (secret) | yes | — | **Injected via the Tailscale Auth Key credential, not sent by the UI directly** |

### Advanced — leave blank/default unless you specifically need to override them

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `storage` | string | no | `""` (auto-detect) | Proxmox storage backend for the VM's disk. Validated against the host every run (`pvesm status --content images`) — blank, or any value that isn't actually valid on this host, is auto-corrected to the first valid one, with a warning in the job output. Only meaningfully overrides anything on a host with more than one valid option. |
| `bridge` | string | no | `""` (auto-detect) | Proxmox network bridge. Validated against the host every run — blank, or any value that doesn't actually exist on this host, is auto-corrected (prefers `vmbr0` if present, otherwise the first bridge found), with a warning in the job output. Only meaningfully overrides anything on a host with multiple bridges. |
| `vm_user` | string | no | `"pronto"` | Login user baked into the VM via cloud-init |
| `tailscale_tag` | string | no | `"tag:vm-provisioned"` | Tailscale ACL tag applied at join time |

### Example `extra_vars`

```json
{
  "os_type": "ubuntu-24.04",
  "vm_name": "app-vm-01",
  "memory": 2048,
  "cores": 2,
  "disk_size": "15G",
  "expose_to_internet": false,
  "expose_port": 80
}
```
(`tailscale_authkey` is not included here — it comes from the attached Credential, not from UI input. `storage`/`bridge`/`vm_user`/`tailscale_tag` are omitted here since their defaults/auto-detection cover the normal case — add them only when overriding.)

### Behavior / things the UI should communicate to the user

- Joining the tailnet requires a one-time manual approval in the Tailscale admin console (unless auto-approval / device-approval-off is configured tailnet-wide) — the job itself finishes quickly regardless of whether approval has happened yet; it does **not** block waiting for it.
- The job's final message tells you whether the VM was already reachable over Tailscale SSH at the time the job finished. If not, that's expected and not a failure — just means approval is still pending.
- If `expose_to_internet: true`, exposure is handled the same detached-background-script way as the other playbooks (delivered via the QEMU guest agent, not SSH, since the VM may not be tailnet-reachable yet at this point) — check `/var/log/tailscale-funnel.log` inside the VM for status. Nothing needs to be run separately for this to happen.
- `os_type` is deliberately limited to Ubuntu/Debian — RHEL-family images (CentOS/Rocky/AlmaLinux) were tried and removed due to unresolved boot/cloud-init issues on this Proxmox host.
- `storage`/`bridge` are both validated against the host every run and auto-corrected if wrong or unset — the caller never needs to know that specific host's storage layout or bridge naming just to launch a VM, and a stale/wrong saved value (e.g. an old AWX Survey default) can't silently break a launch either.
- **Idempotent by `vm_name`, and self-healing like Create LXC.** If a template or VM already exists by name but never actually finished (e.g. an earlier run failed partway through and left a shell with no real disk or no guest agent enabled), it's automatically destroyed and rebuilt rather than reused — confirmed live, this exact situation happened and previously produced VMs that could never boot (no disk at all) until this check was added. You should never need to manually `qm destroy` a broken template/VM from a failed run before relaunching.
- **Verified working side by side on both amd64 and arm64 Proxmox hosts** — template build (including the architecture-correct cloud image), disk import/attach, cloud-init (the VM's boot disk controller differs by architecture — arm64 needs the cloud-init drive on `scsi1`, amd64 uses `ide2` — handled automatically), clone, and boot all complete correctly on each, using the exact same `extra_vars`.

---

## 3. Create LXC — `Create_LXC_App.yml`

Creates an LXC container directly on Proxmox (no VM layer), installs a chosen app inside it, and joins it to the tailnet. Re-running with the same `ct_name` reuses the existing container (e.g. to flip `expose_to_internet` on later) instead of creating a duplicate.

**Two ways to put an app in the container — pick one per launch:**
1. **Native** (`app_choice`/catalog, described below) — the container's only job is running that one app directly on the OS. Default behavior, unchanged.
2. **Docker** (`docker_apps`, same schema as [Install Docker And App](#4-install-docker-and-app--install_docker_and_appyml)) — the container runs Docker instead, and you deploy any number of containers into it by image name, exactly like the VM playbook already does. **This is the one that needs zero code changes for a genuinely new app** — anything with a published Docker image (which is nearly everything) just needs an image name, no catalog entry, no custom install script, ever. Set `docker_apps` to a non-empty list to use this path; leave it empty (default) for native installs. See section 4 below for the full field list, catalog presets, and exposure model — it's identical here.

**Connects to:** the Proxmox host directly (`pct`, `pveam`, `pvesh` commands) for container creation; if `docker_apps` is set, also directly over SSH to the container itself (over the tailnet) to install Docker and deploy the apps — the same connection pattern Install Docker And App uses for a VM. This means the `docker_apps` path needs the container to already be Tailscale-approved before it can proceed (unlike the native path, which uses `pct exec` from the Proxmox host and never needs tailnet connectivity at all) — if approval is still pending, that step fails clearly rather than hanging; re-run once approved.

### Simple — the UI's default form should only need these

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `ct_name` | string | yes | `"app-lxc01"` | Container name and tailnet hostname |
| `app_choice` | string | no | `"nginx"` | A name from the built-in app catalog (currently `nginx`, `ollama`), or any apt package name for anything not in the catalog. A UI should present the catalog names as a dropdown/picker — no shell knowledge needed for those. |
| `memory` | integer | no | `1024` | RAM in MB — for `ollama`, use at least `4096` |
| `cores` | integer | no | `1` | CPU cores |
| `ct_disk_gb` | integer | no | `8` | Root filesystem size in GB (plain number, no unit suffix) — for `ollama`, use at least `20` |
| `expose_to_internet` | boolean | no | `false` | If true, exposes the app publicly via Tailscale Funnel once approved |
| `tailscale_authkey` | string (secret) | yes | — | Injected via the Tailscale Auth Key credential |

### App catalog (built-in presets)

| `app_choice` value | What happens | Default `expose_port` |
|---|---|---|
| `nginx` | Installed via apt | `80` |
| `ollama` | Installed via Ollama's own installer (handles the `zstd` dependency automatically), then pulls `llama3.2:1b` so it's immediately usable | `11434` |
| anything else | Installed via apt using `app_choice` as the package name (e.g. `redis-server`, `postgresql`) | `80` |

`expose_port` is filled in automatically from this table and never needs to be specified for a catalog app — only set it explicitly to override.

### Advanced — leave blank/default unless you specifically need to override them

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `os_template` | string | no | `"ubuntu-24.04-standard"` | Proxmox LXC appliance template name (no version/arch suffix) |
| `storage` | string | no | `""` (auto-detect) | Block storage for the container's rootfs. Validated against the host every run — blank, or any value that isn't actually valid on this host, is auto-corrected to the first valid one supporting `rootdir` content, with a warning in the job output. |
| `template_storage` | string | no | `""` (auto-detect) | Storage for the LXC template file. Validated against the host every run — blank, or any value that isn't valid, is auto-corrected to the first one supporting `vztmpl` content. |
| `bridge` | string | no | `""` (auto-detect) | Proxmox network bridge. Validated against the host every run — blank, or any value that doesn't actually exist on this host, is auto-corrected (prefers `vmbr0` if present, otherwise the first bridge found). |
| `ct_nameserver` | string | no | `"1.1.1.1"` | DNS resolver (unprivileged LXC containers don't reliably get DNS via DHCP) |
| `ct_ip` | string | no | `""` | Leave blank (recommended for everyone, including admins). DHCP is tried first; if it genuinely fails, the job auto-detects the host's subnet/gateway and picks a free static IP itself — no one needs to supply network details. Only set this to force one specific known-good address. Must be set together with `ct_gateway`. |
| `ct_gateway` | string | no | `""` | Gateway IP for the subnet in `ct_ip`. Required together with `ct_ip`; the job fails fast with a clear message if only one of the two is set. |
| `app_install_script` | string | no | `""` | Escape hatch for a one-off app not worth adding to the catalog — any shell command, overrides both the catalog and the plain apt install entirely. Requires real shell/install knowledge; don't expose this as a plain text field to a normal end user — if you need it often enough, add a catalog entry in the playbook instead. |
| `app_post_install_script` | string | no | `""` | Shell command that runs once after install, alongside `app_install_script` above. Same caveat — power-user field, not normal-user input. |
| `docker_apps` | array of objects | no | `[]` | Switches this container to the Docker deployment path — same schema as [Install Docker And App's `docker_apps`](#4-install-docker-and-app--install_docker_and_appyml) (catalog presets, custom images, exposure model, all identical). Non-empty means Docker instead of a native app; `app_choice`/`app_install_script`/`app_post_install_script`/the native exposure block are all skipped. This is the path to use for a genuinely new app with no catalog entry — pass an image name, no code change needed. |
| `wordpress_db_password` | string (secret) | conditional | `""` | Required only if `wordpress` is one of the `docker_apps` entries |
| `gpu_passthrough` | boolean | no | `false` | Passes the Proxmox host's DRM render nodes (`/dev/dri`) into the container, and additionally its Mali GPU (`/dev/mali0`) if present — both detected automatically each run, skipped harmlessly (with a warning in the job output) on a host with neither. The `/dev/dri` half is generic, not ARM-specific — confirmed live: an amd64 host with its own (non-Mali) GPU got its DRI devices passed through correctly too, side by side with a Mali-equipped ARM host in the same run. Only the `/dev/mali0` detection is CIX/ARM-specific; NVIDIA/AMD-specific passthrough (e.g. `/dev/nvidia*`) isn't implemented. Getting a device into the container is not the same as an app being able to use it for acceleration — that also needs a matching userspace driver inside the container (a Mali Vulkan driver, in the ARM case), which this does not install. |
| `tailscale_tag` | string | no | `"tag:lxc-host"` | Tailscale ACL tag applied at join time |

### Example `extra_vars` — simple case (plain nginx)

```json
{
  "ct_name": "app-lxc-01",
  "app_choice": "nginx",
  "memory": 1024,
  "cores": 1,
  "ct_disk_gb": 8,
  "expose_to_internet": false
}
```
(`storage`/`template_storage`/`bridge`/`expose_port` etc. are all omitted — auto-detection and the app catalog cover the normal case entirely.)

### Example `extra_vars` — Ollama, still just catalog fields, no shell knowledge needed

```json
{
  "ct_name": "ollama-lxc01",
  "app_choice": "ollama",
  "memory": 4096,
  "cores": 4,
  "ct_disk_gb": 20,
  "expose_to_internet": false
}
```
The install script, post-install model pull, and `expose_port: 11434` all come from the catalog automatically — a normal user only ever picked `"ollama"` from a list.

### Example `extra_vars` — advanced case (GPU passthrough)

```json
{
  "ct_name": "ollama-gpu-lxc01",
  "app_choice": "ollama",
  "memory": 4096,
  "cores": 4,
  "ct_disk_gb": 20,
  "gpu_passthrough": true,
  "expose_to_internet": false
}
```
`ct_ip`/`ct_gateway` aren't needed here either, even on a host with an unreliable DHCP pool — the job auto-recovers on its own (see behavior notes below). They're left in the Advanced table only as a rare explicit override, not something this example needs.

### Example `extra_vars` — Docker path (any image, zero code changes)

```json
{
  "ct_name": "docker-lxc01",
  "memory": 2048,
  "cores": 2,
  "ct_disk_gb": 16,
  "docker_apps": [
    { "name": "nginx", "path_prefix": "/" }
  ],
  "expose_to_internet": true
}
```
`app_choice` is omitted entirely here — `docker_apps` being non-empty is itself what switches this container from "run one native app" to "run Docker, deploy these images." Swap `docker_apps` for anything else with a published image (`ollama/ollama`, `localai/localai`, a database, a custom app) and it works identically — see section 4 for the full field list and catalog presets.

### Behavior notes

- Same tailnet-approval caveat as Create VM: the job doesn't block waiting for approval.
- If `expose_to_internet: true`, a background script inside the container polls for approval and enables Funnel on its own once approved — check `/var/log/tailscale-funnel.log` inside the container for status. Only useful for services on `expose_port` that speak HTTP.
- Adding a genuinely new app to the catalog (not just using `app_choice` as a raw apt package name) is a data-only edit to `lxc_app_catalog` in the playbook — no task logic to touch. `app_install_script`/`app_post_install_script` remain available directly for a true one-off that isn't worth cataloging; both run verbatim inside the container as root, so treat them as trusted operator/admin input, same as `tailscale_authkey` — never expose them as free-text fields to a normal end user.
- `gpu_passthrough: true` gets the Mali/DRI devices into the container automatically — no manual `.conf` editing, ever. Every prior manual attempt at this on the Orange Pi host is what caused a container to fail to start (`newgidmap` rejecting a custom identity GID mapping); this playbook's version deliberately avoids that by relying on world-writable device bind-mount permissions instead.
- **Idempotent by `ct_name`** (matching Create VM's behavior): if a container with that name already exists, this job skips template staging/creation entirely and reuses it — it only re-checks tailnet reachability and (re-)applies `expose_to_internet`. This is the intended way to make an already-running container public later: re-launch with the same `ct_name` and `expose_to_internet: true`, don't create a second container.
- **A relaunch self-repairs a container left broken by a prior interrupted run.** Only template staging/`pct create`/TUN/GPU config are skipped for an existing container by name — starting the container if stopped, network readiness (DHCP + static-IP fallback), app install, and the Tailscale join all run on *every* invocation, not just for brand-new containers. Confirmed live: an app install that got stuck mid-way while the network was down left a container half-configured (service never started) with the job still reporting success; relaunching the same job now re-verifies and fixes that instead of silently trusting the container's name. You only need to `pct stop <ctid> && pct destroy <ctid>` manually if the container itself is fundamentally broken (e.g. wrong rootfs) — a stalled install or network is fixed by just relaunching.
- **DHCP failure self-heals automatically, no `ct_ip`/`ct_gateway` needed.** The job tries DHCP first; if a container's `eth0` never gets an IPv4 address after retrying, it auto-detects this host's own subnet/gateway (from its routing table), probes a handful of addresses near the top of that range, and applies the first free one as a static IP — confirmed as a real failure mode live (a container's DHCP requests reached the bridge fine but got zero responses because the router's pool was full). `ct_ip`/`ct_gateway` still exist to force one specific known-good address, but nobody — admin or end user — needs to supply network details for the normal/recovery path.
- `storage`/`template_storage`/`bridge` are all validated against the host every run and auto-corrected if wrong or unset — this is deliberately not just a "blank means auto-detect" check. A stale AWX Survey default (confirmed live: `storage: local-lvm` kept getting sent from an old saved value on a host that doesn't have `local-lvm` at all) is treated exactly like an unset field — the playbook always checks the real host and corrects to a valid answer rather than trusting whatever value it was handed. A warning appears in the job output whenever a correction happens, showing what was requested vs. what's actually available.
- **`docker_apps` (non-empty) is the path for a genuinely new app that isn't a plain apt package** — no catalog entry needed, ever, unlike `app_install_script`. Reuses the exact same deployment logic (`deploy_docker_apps_block.yml`) as Install Docker And App, over SSH to the container itself once it's Tailscale-approved. **Not yet live-tested end to end** at the time this was written — the mechanics (shared task file, `add_host`/`hosts` conditional-play pattern, Tailscale SSH reachability) are all individually proven elsewhere in this repo, but this specific combination is new.
- **Verified working side by side on both amd64 and arm64 Proxmox hosts** — template selection/download, DHCP (with the static-IP self-heal available if it's ever needed), and GPU passthrough (correctly discriminating real hardware per host, not architecture) have all been confirmed live with the exact same `extra_vars` launched against both at once.

---

## 4. Install Docker And App — `Install_Docker_And_App.yml`

The most flexible playbook. Installs Docker + Traefik on an existing VM (created by Create VM above), then deploys any number of containers on it — databases, web servers, or fully custom images — with `docker run`-equivalent flexibility (custom image, command, env vars, volumes, extra ports) plus a clean exposure model layered on top.

**Connects to:** the target VM directly over SSH (not the Proxmox host).

### Top-level variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `target_host` | string | yes | `"dev-vm01"` | The VM's tailnet hostname (the `vm_name` from Create VM) |
| `vm_user` | string | no | `"pronto"` | SSH user on the VM |
| `docker_apps` | array of objects | yes | `[{name: nginx, path_prefix: "/"}]` | The containers to deploy — see schema below |
| `wordpress_db_password` | string (secret) | conditional | `""` | Required only if `wordpress` is one of the `docker_apps` |
| `expose_to_internet` | boolean | no | `false` | Exposes every HTTP app in `docker_apps` publicly via one shared Tailscale Funnel URL, once approved |
| `expose_port` | integer | no | `80` | Informational only in the current design (Traefik always listens on 80 internally) |

### `docker_apps` entry schema

Each item in the `docker_apps` array is one container. Every field is optional except `name`; anything you don't set falls back to a catalog preset's default (see the catalog table below) if `name` matches one, or must be supplied explicitly for a fully custom app.

| Field | Type | Description |
|---|---|---|
| `name` | string | **Required.** A catalog preset name (`nginx`, `wordpress`, `portainer`, `redis`, `mongodb`) or any custom name you choose. Used as the container name and the `/opt/<name>` directory on the VM. |
| `image` | string | Docker image, e.g. `"redis:latest"` or `"myregistry.example.com/my-app:1.2.3"`. Required for a non-catalog name; overrides the catalog preset's image if both are given. |
| `port` | integer | The port the app listens on *inside* its container. Required for a non-catalog name. |
| `command` | array of strings | Command override, `docker run`-style, e.g. `["redis-server", "--requirepass", "mypassword"]`. |
| `environment` | object | Environment variables, e.g. `{"MONGO_INITDB_ROOT_USERNAME": "admin", "MONGO_INITDB_ROOT_PASSWORD": "mypassword"}`. |
| `volumes` | array of strings | `docker run -v`-style mounts: `"source:target"` or `"source:target:mode"`. A `source` that isn't an absolute/relative path (e.g. `"mydata:/data"`) is treated as a named volume and created automatically. |
| `ports` | array of strings | **Extra** host:container port publishes beyond what `expose_http`/`tcp_funnel_port` already handle, `docker run -p`-style: `"6379:6379"` or `"127.0.0.1:9000:9000"`. |
| `extra_compose` | object | Raw dict merged directly into the generated compose service definition — an escape hatch for anything not covered above (`restart`, `user`, `privileged`, `entrypoint`, `healthcheck`, `cap_add`, etc). |
| `expose_http` | boolean | Whether this app gets an HTTP route through Traefik. Defaults to the catalog preset's value, or `true` for a custom app — **except** it defaults to `false` automatically whenever `tcp_funnel_port` is set (a TCP service shouldn't also become an HTTP router). |
| `path_prefix` | string | The URL path this app answers on behind Traefik, e.g. `"/"` or `"/blog"`. Only meaningful when `expose_http` resolves to true. All HTTP apps share one Traefik instance and one public URL, distinguished by path. |
| `tcp_funnel_port` | integer | One of `443`, `8443`, `10000`. If set, this app is exposed to the **public internet as a raw TCP service** (not HTTP) via Tailscale Funnel, independent of `expose_to_internet` above — setting this field is itself the opt-in for this one app. Required for any non-HTTP service (database) you want reachable from outside the tailnet. **You are responsible for setting real authentication via `command`/`environment` — never set this on a database with no auth configured.** |

### Docker app catalog (built-in presets)

| Preset name | Image | Port | HTTP by default? | Notes |
|---|---|---|---|---|
| `nginx` | `nginx:latest` | 80 | yes | — |
| `wordpress` | `wordpress:latest` | 80 | yes | Deploys its own MySQL sidecar automatically; needs `wordpress_db_password` |
| `portainer` | `portainer/portainer-ce:latest` | 9000 | yes | Mounts the Docker socket + a persistent volume automatically |
| `redis` | `redis:latest` | 6379 | no | Internal-only by default — no auth is set unless you add `command` |
| `mongodb` | `mongo:latest` | 27017 | no | Internal-only by default, with a persistent volume automatically |

### Exposure model — three independent modes per app

1. **Internal-only** (default for `redis`/`mongodb`, or any app with neither `expose_http` nor `tcp_funnel_port` set): reachable only by other containers on the same VM, by container name (e.g. `redis:6379`). Not reachable from the tailnet or the internet at all.
2. **HTTP via Traefik** (`expose_http: true` + `path_prefix`): reachable at `http://<target_host>/<path_prefix>` over the tailnet always, and additionally at `https://<target_host>.<tailnet-name>.ts.net/<path_prefix>` from the **public internet** once `expose_to_internet: true` is set on the top-level playbook run and tailnet approval completes.
3. **Raw TCP via Funnel** (`tcp_funnel_port` set): reachable from the **public internet** directly at `<target_host>.<tailnet-name>.ts.net:<tcp_funnel_port>`, using TLS at the transport level (the client must connect with TLS even though the backend service itself doesn't speak TLS — e.g. `redis-cli --tls --sni <host> ...`, or a MongoDB URI with `&tls=true`). Independent of `expose_to_internet`.

There's also a fourth pattern the schema supports but has no dedicated field for — **tailnet-only reachable, not public** — achieved by adding a plain `ports` entry (not `tcp_funnel_port`) without Tailscale Funnel involved at all:
```json
{ "name": "redis-tailnet", "image": "redis:latest", "port": 6379, "expose_http": false,
  "command": ["redis-server", "--requirepass", "mypassword"], "ports": ["6379:6379"] }
```
This publishes the port to all of the VM's interfaces (reachable by anything on the tailnet or same LAN as the VM, not the public internet).

### Example `extra_vars` — mixed deployment (public web app + two internal DBs + one public DB)

```json
{
  "target_host": "app-vm-01",
  "expose_to_internet": true,
  "expose_port": 80,
  "docker_apps": [
    { "name": "nginx", "path_prefix": "/" },
    { "name": "redis" },
    { "name": "mongodb" },
    {
      "name": "mongodb-external",
      "image": "mongo:latest",
      "port": 27017,
      "expose_http": false,
      "environment": {
        "MONGO_INITDB_ROOT_USERNAME": "admin",
        "MONGO_INITDB_ROOT_PASSWORD": "REPLACE_WITH_STRONG_PASSWORD"
      },
      "volumes": ["mongodb_external_data:/data/db"],
      "tcp_funnel_port": 8443
    }
  ]
}
```

### Behavior notes for the UI

- Re-running this playbook is safe/idempotent — Traefik and any already-correct app are left alone (`docker compose up -d` no-ops on unchanged config); only apps whose entry actually changed get recreated.
- If a container is deleted and immediately recreated with the same image on the same host, you may see a transient "Gateway Timeout" through Traefik for a minute or two afterward (a known Docker bridge-networking quirk, not a bug) — this self-resolves; no action needed.
- Real failures (bad image, port already in use, etc.) surface as a full, untruncated Ansible task failure with the actual Docker error message — the UI should just display the job's failure message directly, no need to dig deeper.
- Two different apps cannot use the same `tcp_funnel_port` or the same host port in `ports` on the same VM — validate this client-side if possible before submitting, since it will fail with a clear "port already allocated" error otherwise.

---

## 5. Expose App To Internet — `Expose_App_To_Internet.yml`

A standalone playbook for exposing something on a VM *after the fact* — e.g. a manually-installed `apt` package, an app added outside `Install_Docker_And_App.yml`'s schema, or an app on a VM that was created without `expose_to_internet: true` at the time. Purely additive: it doesn't know or care how the target port got a listener.

Note: as of the `expose_to_internet` addition to Create VM/Create LXC, this playbook is **no longer needed for the common case** of "expose a VM right after creating it" — that's now handled in one job. It's still the right tool for exposing something added to an already-running VM later, or for a VM/LXC that was deliberately created private and only later needs to go public.

**Connects to:** the target VM directly over SSH.

### Variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `target_host` | string | yes | `"dev-vm01"` | The VM's tailnet hostname |
| `vm_user` | string | no | `"pronto"` | SSH user on the VM |
| `expose_port` | integer | no | `80` | The local port already listening on the VM to expose via Tailscale Funnel |

### Example `extra_vars`

```json
{
  "target_host": "app-vm-01",
  "vm_user": "pronto",
  "expose_port": 80
}
```

### Behavior notes

- HTTP-only — this uses the same `tailscale serve`/`funnel` mechanism as `Install_Docker_And_App.yml`'s `expose_to_internet`, so `expose_port` must be an HTTP service.
- Same approval-polling behavior: doesn't block, hands off to a background script, check `/var/log/tailscale-funnel.log` on the VM for status.

---

## 6. Delete VM — `Delete_VM.yml`

Destroys a VM (or a template — Proxmox templates are just specially-flagged VMs, so this same playbook works for both).

**Connects to:** the Proxmox host directly.

### Variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `vm_name` | string | yes | `"dev-vm01"` | Name of the VM (or template) to delete |

### Example `extra_vars`

```json
{ "vm_name": "app-vm-01" }
```

### Behavior notes

- Safe to call on a name that doesn't exist — reports "nothing to delete" and exits cleanly rather than failing.
- If you want a full "rebuild from scratch" flow in the UI (e.g. to pick up an updated Proxmox VM template), you need **two** calls: one with the VM's own name, and a separate one with the underlying template's name (e.g. `ubuntu-24.04-template`) — deleting only the VM leaves the template (and whatever's baked into it) untouched.

---

## 7. Delete LXC — `Delete_LXC.yml`

Destroys an LXC container.

**Connects to:** the Proxmox host directly.

### Variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `ct_name` | string | yes | `"app-lxc01"` | Name of the LXC container to delete |

### Example `extra_vars`

```json
{ "ct_name": "app-lxc-01" }
```

### Behavior notes

- Same "safe no-op if it doesn't exist" behavior as Delete VM.

---

## 8. Delete Docker App — `Delete_Docker_App.yml`

Removes one or more Docker apps (by their `docker_apps` entry `name`) from a VM — stops and removes the containers, networks, and volumes for just those apps, and deletes their `/opt/<name>` directory. Traefik and every other app on the same VM are left running untouched.

**Connects to:** the target VM directly over SSH.

### Variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `target_host` | string | yes | `"dev-vm01"` | The VM's tailnet hostname |
| `vm_user` | string | no | `"pronto"` | SSH user on the VM |
| `app_names` | array of strings | yes | `["nginx"]` | The app name(s) to remove — matches the `name` field used when it was deployed via `Install_Docker_And_App.yml` |

### Example `extra_vars`

```json
{
  "target_host": "app-vm-01",
  "vm_user": "pronto",
  "app_names": ["redis-external", "mongodb-external"]
}
```

### Behavior notes

- `app_names` is always a list, even for deleting just one app (`["nginx"]`).
- Safe to include a name that isn't actually deployed — reports "nothing to delete" for that one and continues with the rest of the list.
- Does not touch Traefik's own `/opt/traefik` — Traefik itself has no dedicated delete playbook today; if you ever need to remove it, that would need a manual `docker compose down` on the VM or a new playbook.

---

## Appendix A — full `os_image_lookup` catalog (for Create VM's `os_type`)

| `os_type` value | Description |
|---|---|
| `ubuntu-24.04` | Ubuntu 24.04 LTS (Noble) |
| `ubuntu-22.04` | Ubuntu 22.04 LTS (Jammy) |
| `debian-12` | Debian 12 (Bookworm) |

RHEL-family images (CentOS Stream 9, Rocky 9, AlmaLinux 9) are intentionally not offered — they hit unresolved boot/cloud-init issues on this Proxmox host and were removed from the catalog.

## Appendix B — operational gotchas worth surfacing in the UI

- **Tailnet device approval** is the one step that can't be automated away entirely (unless your Tailscale tailnet has "Require device approval" turned off, or `autoApprovers` configured). Any Create playbook, and any exposure action, may sit waiting on this. Surface a clear "pending tailnet approval" state in the UI rather than treating a slow job as stuck.
- **`tcp_funnel_port` requires real authentication.** The UI should not let a user set `tcp_funnel_port` on a database entry without also requiring a `command`/`environment` field that sets a password — there is no other protection layer.
- **Password characters in connection strings**: if a user-supplied password contains `@`, `:`, or other URI-special characters, any connection string the UI generates for MongoDB (or similar `scheme://user:pass@host` formats) must URL-encode the password (`@` → `%40`, etc.) or the connection string will fail to parse.
- **Job Template "Prompt on Launch"** must be enabled for Variables on every template the UI calls — this was a real, repeated source of confusion during manual testing (edits at launch time were silently ignored otherwise).
- **`expose_to_internet` on Create VM depends on the QEMU guest agent starting up inside the new VM** (it's delivered via `qm guest exec`, not SSH, since the VM may not be tailnet-reachable yet). The job waits up to 5 minutes for the guest agent before giving up — on a very slow-booting VM this could still be tight; if you ever see this specific step fail, it's almost always the guest agent not being ready in time, not a real Funnel/Tailscale problem.
- **No dynamic inventory today.** Every playbook target (`vm_name`, `ct_name`, `target_host`) is a name the caller must already know — there's no live "list of VMs that currently exist" the UI can query from AWX itself. This is planned as a separate follow-up (likely an AWX Inventory Source using the `community.general.proxmox` plugin), not something covered by anything in this document yet.
- **AWX Project sync isn't automatic.** An AWX Project backed by git only re-pulls when something triggers it — either "Update Revision on Launch" enabled on the Job Template, or a manual/scheduled Project sync. A code change pushed to GitHub can silently have zero effect on real job runs until one of those happens. Also worth knowing: more than one AWX Project can point at the same or related repos and drift out of sync with each other — confirm which Project a Job Template actually uses, and its current revision hash, before assuming a code fix has taken effect.
- **Maintainer gotcha: `set_fact` can never override a variable that also arrived as an `extra_var`** (which is exactly what every AWX Survey answer and Job Template "Variables" field becomes). Extra vars have the highest precedence in Ansible — above `set_fact`, above everything — so a task that tries to "correct" or fill in a variable using its own name (e.g. `set_fact: storage: "{{ ... }}"` when `storage` is also a Survey field) is silently ignored the moment that variable is referenced again; the original extra_var value always wins. Confirmed live: this exact mechanism was why the storage/bridge/app-catalog auto-correction logic in Create VM and Create LXC didn't work on the first several attempts, despite the logic itself being correct — the fix was to compute a **distinctly-named** fact (`resolved_storage`, `resolved_bridge`, `resolved_app_install_script`, `resolved_expose_port`, etc.) and use only that name downstream, never reassigning the original Survey variable name. Apply the same pattern to any future "validate and auto-correct" logic added to these playbooks.
- **A brand-new host needs working SSH + passwordless sudo (or a direct root credential) before AWX can do anything with it at all** — this is a one-time bootstrap prerequisite, not something any playbook can configure remotely (you can't use a broken sudo to fix a broken sudo). Confirmed live: a host whose SSH user lacked `NOPASSWD` sudo failed every job at the very first task ("Gathering Facts") with `"Missing sudo password"`, before any real playbook logic ever ran. Bake this into whatever process onboards a new Proxmox host (e.g. `post_boot.sh`) so it's never a surprise per-host.
- **Prefer stable Tailscale DNS hostnames over raw IPs in AWX's inventory.** A device's Tailscale IP can change if it re-registers — confirmed live: a host's inventory entry silently went stale after its IP changed, causing SSH timeouts on every job that looked exactly like a playbook bug but wasn't. The `<device-name>.<tailnet-name>.ts.net` hostname stays constant even if the underlying IP changes later.
- **A small/resource-constrained Proxmox host can hit genuine OOM kills under concurrent load** — confirmed live: the Linux kernel killed a running VM's QEMU process outright (`Out of memory: Killed process ... (kvm)`) on a board with only ~7.4GB RAM while a Docker install and other test containers were running at the same time. This isn't a playbook bug on any architecture, and isn't fixable in the playbook — it's a real hardware capacity limit. On a constrained host, test one heavy workload at a time and check `free -h`/`pct list`/`qm list` for what else is already running before launching another.
- **`gpu_passthrough`'s `/dev/dri` detection is generic, not ARM-specific** — confirmed live: an amd64 host with its own (non-Mali) GPU had its DRI devices passed through correctly too, right alongside a Mali-equipped ARM host in the same run. Don't assume a host is GPU-less just because it isn't the ARM one you had in mind.
