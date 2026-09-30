# Parameters

Full reference for every variable this role reads, with defaults and example values.

## [defaults/main.yml](../defaults/main.yml)

```yaml
usecase: "install"    # available usecases
# - install - default installation
# - get_kubeconfig - only get kubeconfig

microk8s_plugins:
  enabled: true         # feature toggle
  dns: true              # CoreDNS
  ingress: true          # Ingress controller for external access
  dashboard: true        # The Kubernetes dashboard

ufw:                    # Setup UFW when installed
  enabled: true          # feature toggle
  kubectl_port: 16443    # kubectl port
  kubectl_allow_from: "" # restrict kubectl_port to this source (CIDR/IP) - empty = open to everyone
  ingress_port:          # ingress ports
    - 443

route_service:          # routing 10.152.183.0/24 (MicroK8s Service CIDR) via a dedicated dummy interface
  enabled: true                # feature toggle
  interface: microk8s-svc0     # name of the dummy interface this role creates

get_kubeconfig:
  enabled: true         # feature toggle
```

## [vars/main.yaml](../vars/main.yaml)

```yaml
exist_usecases:
  - "install"
  - "get_kubeconfig"
```

Not meant to be overridden - this is the allow-list `usecase` is validated against in [tasks/main.yml](../tasks/main.yml), not a user-facing setting.

## Top-level variables

| Variable | Type | Default | Where it's used |
| --- | --- | --- | --- |
| `usecase` | `str` | `install` | [tasks/main.yml](../tasks/main.yml) - selects which task lists run this invocation. |
| `microk8s_plugins` | `dict` | see above | [tasks/setup_plugins.yaml](../tasks/setup_plugins.yaml) |
| `ufw` | `dict` | see above | [tasks/setup_ufw.yaml](../tasks/setup_ufw.yaml) |
| `route_service` | `dict` | see above | [tasks/setup_route_service.yaml](../tasks/setup_route_service.yaml), [templates/route-service.j2](../templates/route-service.j2) |
| `get_kubeconfig` | `dict` | `{enabled: true}` | [tasks/main.yml](../tasks/main.yml) - whether the `install` usecase also fetches a kubeconfig at the end. |
| `microk8s_api_host` | `str` | `""` | [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) - overrides the API address written into the saved kubeconfig. |
| `microk8s_api_sans` | `list[str]` | `[]` | [tasks/setup_api_sans.yaml](../tasks/setup_api_sans.yaml) - extra SANs for the kube-apiserver certificate. |

## `usecase`

Selects which task lists [tasks/main.yml](../tasks/main.yml) runs. Must be one of `exist_usecases` (see above) or the role fails an assert immediately.

| Value | What runs |
| --- | --- |
| `install` | Everything: install MicroK8s, `ufw` (if `ufw.enabled`), `route_service` (if `route_service.enabled`), wait for the cluster to be ready, `microk8s_plugins` (if `microk8s_plugins.enabled`), then kubeconfig fetch (if `get_kubeconfig.enabled`). |
| `get_kubeconfig` | Only [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) - re-fetch the kubeconfig without touching the cluster. Use this to refresh `~/.kube/kubeconfig_<host>` after the cluster's certs were regenerated, without re-running the full install. |

```yaml
- hosts: microk8s_hosts
  roles:
    - role: InsonusK.MicroK8S
      vars:
        usecase: get_kubeconfig
```

## `microk8s_plugins`

Reconciles MicroK8s addons via `microk8s.enable`/`microk8s.disable`, driven by [tasks/setup_plugins.yaml](../tasks/setup_plugins.yaml). Only runs when `microk8s_plugins.enabled: true`.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `bool` | `true` | Feature toggle for the whole addon-reconciliation step. |
| `dns` | `bool` | `true` | CoreDNS addon. |
| `ingress` | `bool` | `true` | nginx ingress controller addon. |
| `dashboard` | `bool` | `true` | Kubernetes dashboard addon. |

Any other MicroK8s addon name also works as a key - the task loops over every addon `microk8s status --format yaml` reports and only acts on the ones present in `microk8s_plugins`, so addons you don't list are left exactly as they are (never toggled by this role). A value that isn't exactly `true` is passed through as an argument (`microk8s.enable <name>:<value>`):

```yaml
microk8s_plugins:
  enabled: true
  dns: true
  ingress: true
  dashboard: false          # explicitly disabled - the role will call microk8s.disable
  metallb: "10.0.0.1-10.0.0.10"   # enabled with an argument
```

## `ufw`

Firewall rules for MicroK8s, applied by [tasks/setup_ufw.yaml](../tasks/setup_ufw.yaml). Only runs when `ufw.enabled: true`.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `bool` | `true` | Feature toggle. |
| `default_incoming` | `str` | `deny` | Default UFW policy for incoming traffic - set explicitly rather than relying on whatever policy UFW already happened to have. |
| `ssh_port` | `int` | `22` | Rate-limited (`ufw limit`, not just `allow`) - brute-force protection. This is what keeps `Enable UFW` from locking out SSH once `default_incoming: deny` is in effect. |
| `kubectl_port` | `int` | `16443` | Opened for the kube-apiserver (kubectl access). |
| `kubectl_allow_from` | `str` (CIDR/IP) | `""` | Restricts `kubectl_port` to this source. Empty string = open to any source (`ufw`'s `from_ip: any`) - the original, unrestricted behavior. |
| `ingress_port` | `list[int]` | `[443]` | Opened for each listed ingress port, unrestricted (any source). |

The task list also unconditionally allows inbound traffic on the `cni0` interface and routed traffic between MicroK8s' pod network and the rest of the host, regardless of `kubectl_port`/`ingress_port`.

**Setting `ufw` at all replaces the whole dict** (no deep-merge across defaults/group_vars/host_vars - see [doc/setup.md](./setup.md)), so an override that only sets e.g. `kubectl_allow_from` loses `default_incoming`/`ssh_port`/everything else silently. Always spell out every key you care about, even ones equal to the default.

kube-apiserver authenticates clients via mTLS regardless of `kubectl_allow_from` - an open `kubectl_port` doesn't grant access without a valid client cert. `kubectl_allow_from` is defense in depth (keeps the port itself from being reachable/probeable outside the trusted source), not the only thing standing between the API and the internet.

```yaml
ufw:
  enabled: true
  kubectl_allow_from: "10.8.1.0/24"   # e.g. a VPN tunnel subnet - restrict kubectl access to it
  kubectl_port: 16443
  ingress_port:
    - 80
    - 443
```

## `route_service`

Works around pod/service networking breaking when a VPN client runs on the same host as MicroK8s (the VPN's own routing can drop or shadow the route for MicroK8s' Service CIDR, `10.152.183.0/24` by default). Applied by [tasks/setup_route_service.yaml](../tasks/setup_route_service.yaml) + [templates/route-service.j2](../templates/route-service.j2). Only runs when `route_service.enabled: true`.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `bool` | `true` | Feature toggle. |
| `interface` | `str` | `microk8s-svc0` | Name of a **dummy** network interface this role creates and anchors the route to - not an existing physical/VPN NIC. |

The role installs and enables a systemd unit (`add-route.service`) that:

1. Creates a dummy interface named `route_service.interface` (`ip link add ... type dummy`), idempotently.
2. Brings it up.
3. Points the Service CIDR at it (`ip route replace 10.152.183.0/24 dev <interface> proto static`).

Using a self-created dummy interface - rather than an existing LAN NIC name like `enp0s3` - means this works identically on every host regardless of what the real NIC happens to be named (interface naming is not guaranteed to be stable across VM re-creation on all hypervisors), and it correctly reflects that the Service CIDR is a virtual, NAT-only range that was never meant to be transmitted over a real link in the first place.

```yaml
route_service:
  enabled: true
  interface: microk8s-svc0
```

If your cluster's Service CIDR isn't the MicroK8s default (`10.152.183.0/24` - check `microk8s kubectl cluster-info dump | grep -i service-cluster-ip-range` or `/var/snap/microk8s/current/args/kube-apiserver`), you'll need to edit the CIDR in [templates/route-service.j2](../templates/route-service.j2) directly - it isn't currently exposed as a variable.

## `get_kubeconfig`

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | `bool` | `true` | Whether the `install` usecase also runs [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) at the end. Has no effect under `usecase: get_kubeconfig`, which always fetches the kubeconfig regardless. |

See [doc/setup.md](./setup.md#fetching-a-kubeconfig) for what the fetched file looks like and where it's written.

## `microk8s_api_host`

| Type | Default | Description |
| --- | --- | --- |
| `str` (host/IP) | `""` | Overrides the address written into the saved kubeconfig's `server:` line. Empty = fall back to `ansible_host`, then `inventory_hostname`. |

`ansible_host` is not always the address kubectl should actually use. Some inventories compute `ansible_host` dynamically for Ansible's own SSH/provisioning needs - e.g. a directly-reachable address used because a VPN tunnel to the host may not exist yet - which can differ from the address other clients (kubectl, potentially behind a firewall rule scoped to a VPN subnet) are meant to reach it on. Set `microk8s_api_host` explicitly whenever that's the case:

```yaml
# host_vars/<host>/microk8s_dev.yaml
microk8s_api_host: "{{ connections.vpn_ip.host }}"   # example: a custom per-host "connections" scheme
```

## `microk8s_api_sans`

| Type | Default | Description |
| --- | --- | --- |
| `list[str]` (hostnames/IPv4) | `[]` | Extra Subject Alternative Names for the kube-apiserver certificate on port 16443. Applied under `usecase: install` only; empty = step skipped. |

MicroK8s issues `server.crt` itself from `/var/snap/microk8s/current/certs/csr.conf.template`, covering only `kubernetes.*` names and the host's own IPs. If `microk8s_api_host` is a DNS name, kubectl fails with `x509: certificate is valid for kubernetes, ..., not <name>` until that name is listed here.

[tasks/setup_api_sans.yaml](../tasks/setup_api_sans.yaml) writes each entry as `DNS.<100+i>` (or `IP.<100+i>` for entries matching `^[0-9.]+$`) before the template's `#MOREIPS` marker, and runs `microk8s refresh-certs --cert server.crt` if the template changed. The cluster CA is unchanged, so existing kubeconfigs stay valid. Removing an entry from the list does not remove its line from the template.

```yaml
microk8s_api_host: k8s-dev.example
microk8s_api_sans:
  - "{{ microk8s_api_host }}"
```

## Internal / computed facts

Not role inputs - set by the role itself while it runs. Listed here because they can show up in `-v` output/debugging, not because you should set them.

| Fact | Set in | Description |
| --- | --- | --- |
| `microk8s_api_ip` | [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) | The address substituted for `127.0.0.1` in the saved kubeconfig - `microk8s_api_host` if set, otherwise `ansible_host`, otherwise `inventory_hostname`. |
| `microk8s_status` | [tasks/setup_plugins.yaml](../tasks/setup_plugins.yaml) | Parsed `microk8s status --format yaml` output, used to enumerate addons. |

## Implicit requirements (not role variables)

- **`ansible_user`** - added to the `microk8s` group by [tasks/install_microk8s.yaml](../tasks/install_microk8s.yaml), so it must be set (normal for any Ansible-managed host).
- **`ansible_host`** (inventory) - read by [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) to build a reachable API server address when `microk8s_api_host` isn't set; falls back to `inventory_hostname` if unset too.
- **`HOME` environment variable on the control node** - the kubeconfig is written to `$HOME/.kube/kubeconfig_<inventory_hostname>` on the machine running `ansible-playbook`, not on the target host.
