# Setup

How to invoke this role from a playbook, for each usecase it supports.

## Requirements

- Ubuntu 22.04+ target host.
- `ansible_user` must be a real, sudo-capable user on the target (added to the `microk8s` group by the role).
- For the `get_kubeconfig` step: a `$HOME/.kube/` directory writable by whoever runs `ansible-playbook` (the role creates the file itself, but expects the parent directory or its own write permissions to already be in order).

## A note on overriding dict variables

`microk8s_plugins`, `ufw` and `route_service` are plain dicts, set once in [defaults/main.yml](../defaults/main.yml) - this role does not deep-merge them the way some other roles combine `defaults`/`group_vars`/`host_vars` layers. Ansible's normal variable precedence applies instead: whichever `vars:`/`group_vars`/`host_vars` scope has the highest precedence for e.g. `ufw` **replaces the entire dict**, it does not merge key-by-key with the default. So:

```yaml
# This loses default_incoming/ssh_port/ingress_port/etc. - they become
# undefined, not "whatever the default was".
ufw:
  kubectl_allow_from: "10.8.1.0/24"
```

```yaml
# Correct: repeat every key you want, even ones equal to the default.
ufw:
  enabled: true
  default_incoming: deny
  ssh_port: 22
  kubectl_port: 16443
  kubectl_allow_from: "10.8.1.0/24"
  ingress_port:
    - 80
    - 443
```

## Quick start

Minimal inventory + playbook to install MicroK8s on a host, with every default left as-is.

### 1. Add the host to your inventory

```yaml
# inventory/hosts.yaml
all:
  hosts:
    my-microk8s-host:
      ansible_host: 203.0.113.10
      ansible_user: ubuntu
```

### 2. Write a playbook

```yaml
# install_microk8s.yaml
- hosts: my-microk8s-host
  become: true
  gather_facts: true

  roles:
    - role: InsonusK.MicroK8S
```

No `vars:` needed for a default install - see [doc/parameters.md](./parameters.md) for what the defaults actually do (UFW rules, `dns`/`ingress`/`dashboard` addons, the `route_service` dummy-interface fix, and a kubeconfig fetch at the end).

### 3. Run it

```bash
ansible-playbook -i inventory/hosts.yaml install_microk8s.yaml
```

`gather_facts: true` matters - [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) needs `ansible_host` from the inventory, and the `become: true`/`gather_facts: true` combination is what makes the rest of the role (snap install, UFW, systemd) work without per-task overrides.

## What a full `usecase: install` run does

In order (see [tasks/main.yml](../tasks/main.yml)):

1. Validates `usecase` is a known value.
2. Installs the `microk8s` snap and adds `ansible_user` to the `microk8s` group ([tasks/install_microk8s.yaml](../tasks/install_microk8s.yaml)).
3. Configures UFW, if `ufw.enabled` ([tasks/setup_ufw.yaml](../tasks/setup_ufw.yaml)).
4. Sets up the `route_service` dummy-interface route fix, if `route_service.enabled` ([tasks/setup_route_service.yaml](../tasks/setup_route_service.yaml)).
5. Waits for MicroK8s to report ready ([tasks/assert_is_running.yaml](../tasks/assert_is_running.yaml)).
6. Reconciles addons, if `microk8s_plugins.enabled` ([tasks/setup_plugins.yaml](../tasks/setup_plugins.yaml)).
7. Fetches a kubeconfig, if `get_kubeconfig.enabled` ([tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml)).

Each of steps 3-4 and 6-7 can be turned off independently - see [doc/parameters.md](./parameters.md) for the toggles.

## Fetching a kubeconfig

Either as the last step of a full install (`get_kubeconfig.enabled: true`, the default), or standalone:

```yaml
- hosts: my-microk8s-host
  become: true
  gather_facts: true

  roles:
    - role: InsonusK.MicroK8S
      vars:
        usecase: get_kubeconfig
```

Both paths run the same [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) and produce the same result: a standalone kubeconfig file on the control node, at

```text
$HOME/.kube/kubeconfig_<inventory_hostname>
```

with the API server address rewritten from MicroK8s' own `127.0.0.1:16443` to something reachable from the control node (`ansible_host`, or `inventory_hostname` itself if `ansible_host` isn't set - see [doc/parameters.md](./parameters.md#internal--computed-facts)).

Use it directly with `--kubeconfig`, or export it for the session:

```bash
kubectl --kubeconfig ~/.kube/kubeconfig_my-microk8s-host get nodes
# or
export KUBECONFIG=~/.kube/kubeconfig_my-microk8s-host
kubectl get nodes
```

It is **not** merged into `~/.kube/config` and no context is renamed - see the "Disabled" comments in [tasks/get_kubeconfig.yaml](../tasks/get_kubeconfig.yaml) for why (MicroK8s names cluster/user/context identically on every host, so a naive merge across multiple MicroK8s hosts would silently overwrite one host's entries with another's).

Re-run this usecase any time you need to refresh the file - e.g. after MicroK8s regenerated its certs - without repeating the full install.

## Variants

### Multiple clusters from one playbook run

Nothing special needed - `usecase: get_kubeconfig`/`install` both key the output file off `inventory_hostname`, so looping the play over a group produces one `kubeconfig_<host>` file per host:

```yaml
- hosts: microk8s_hosts   # a group with several hosts
  become: true
  gather_facts: true

  roles:
    - role: InsonusK.MicroK8S
```

### Skipping UFW (e.g. firewall managed by another role)

```yaml
roles:
  - role: InsonusK.MicroK8S
    vars:
      ufw:
        enabled: false
```

### Skipping the route_service fix (no VPN client on this host)

```yaml
roles:
  - role: InsonusK.MicroK8S
    vars:
      route_service:
        enabled: false
```

### Only enabling specific addons

```yaml
roles:
  - role: InsonusK.MicroK8S
    vars:
      microk8s_plugins:
        enabled: true
        dns: true
        ingress: false
        dashboard: false
```

### Install without fetching a kubeconfig

```yaml
roles:
  - role: InsonusK.MicroK8S
    vars:
      get_kubeconfig:
        enabled: false
```
