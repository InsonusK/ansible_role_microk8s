Role InsonusK.MicroK8S
=========

Install [MicroK8S](https://microk8s.io/) to Ubuntu

[Ansible galaxy](https://galaxy.ansible.com/ui/standalone/roles/InsonusK/MicroK8S/install/)

Requirements
------------

Ubuntu 22.04+

Role Variables
--------------

See [doc/parameters.md](doc/parameters.md) for the full variable reference (`usecase`, `microk8s_plugins`, `ufw`, `route_service`, `get_kubeconfig`), with defaults and examples. Raw defaults: [defaults/main.yml](./defaults/main.yml).

Dependencies
------------

None

Documentation
-------------

- [doc/setup.md](doc/setup.md) - how to invoke the role: quick start, the `install`/`get_kubeconfig` usecases, common variants (skip UFW, skip the VPN route fix, pick specific addons).
- [doc/parameters.md](doc/parameters.md) - full parameter reference with example values.

Example Playbook
----------------

```yaml
- hosts: server
  roles:
  - role: InsonusK.MicroK8S
```

See [doc/setup.md](doc/setup.md) for more (fetching a kubeconfig standalone, running against multiple hosts, disabling individual steps).

License
-------

Apache 2.0

Author Information
------------------

[InsonusK](https://github.com/InsonusK)
Inspired by [istvano.microk8s](https://galaxy.ansible.com/ui/standalone/roles/istvano/microk8s/documentation/)
