Role InsonusK.MicroK8S
=========

Install [MicroK8S](https://microk8s.io/) to Ubuntu

[Ansible galaxy](https://galaxy.ansible.com/ui/standalone/roles/InsonusK/MicroK8S/install/)

Requirements
------------

Ubuntu 22.02+

Role Variables
--------------

[from defaults/mail.yml](./defaults/main.yml)
```yaml
---
microk8s_plugins:
  dns: true             # CoreDNS
  ingress: true         # Ingress controller for external access
  dashboard: true       # The Kubernetes dashboard

ufw:                    #Setup UFW when installed
  enabled: true         #feature toggle
  kubectl_port: 16443   #kubectl port
  ingress_port:         #ingress ports
    - 443

route_service:          #routing 10.152.183.0/24 into local network, bug fix when k8s inner trafic goes into VPN
  enabled: true         #feature toggle
  interface: enp0s3     #local network interface
```

Dependencies
------------

None

Example Playbook
----------------

```yaml
- hosts: server
  roles:
  - role: InsonusK.MicroK8S
```

License
-------

Apache 2.0

Author Information
------------------

[InsonusK](https://github.com/InsonusK)
Inspired by [istvano.microk8s](https://galaxy.ansible.com/ui/standalone/roles/istvano/microk8s/documentation/)
