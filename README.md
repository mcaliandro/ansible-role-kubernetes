Ansible role for Kubernetes
=========

Install and configure on-premises Kubernetes clusters.

This role is designed to bootstrap and manage Kubernetes clusters using `kubeadm` method, as well as supporting the customization of Kubernetes components with kubeadm APIs.

Requirements
------------

From [ansible.posix](https://galaxy.ansible.com/ui/repo/published/ansible/posix/) collection (version `2.2.2`):
- [sysctl](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/sysctl_module.html) module.

From [community.general](https://galaxy.ansible.com/ui/repo/published/community/general/) collection (version `13.4.0`):
- [dnf_versionlock](https://docs.ansible.com/projects/ansible/latest/collections/community/general/dnf_versionlock_module.html) module;
- [modprobe](https://docs.ansible.com/projects/ansible/latest/collections/community/general/modprobe_module.html) module.


Role Variables
--------------

A description of the settable variables for this role (see `defaults/main.yml`).

**Role of the Kubernetes node**: specify if the host will serve as a `control_plane` (default) or a `worker` node.
```yaml
kubernetes_role: control_plane
```

**Kubernetes release and version**: specify the release channel and the minor version (this will pin the packages to such version).
```yaml
kubernetes_release_channel: "stable"
kubernetes_version: "1.35"
```

**First control plane**: this host will be used as first control-plane when spawning a new cluster, as well as the control-plane that is responsible for the nodes registration and cluster upgrade procedures.
```yaml
kubernetes_first_controlplane: master1.example.org
```

**Cluster name and DNS**: specify a custom name for the cluster (default, `kubernetes`) and the DNS domain (default `cluster.local`).
```yaml
kubernetes_cluster_name: kubernetes
kubernetes_dns_domain: cluster.local
```

**Networking configurations**: specify the Pod and Service subnets (defaults are fully compatible with Flannel CNI).
```yaml
kubernetes_pod_network: "10.244.0.0/16"
kubernetes_service_network: "10.96.0.0/16"
```

**CNI plugin**: specify which CNI plugin to install automatically.
The supported CNI are: `flannel` (default), and `calico`.
You can also specify `none` in case you want to install the CNI manually.
```yaml
kubernetes_cni_plugin: flannel
```

**CNI plugin version**: specify which CNI plugin version to install.

**NOTE**: this variable is required for Calico installation.
```yaml
kubernetes_cni_plugin_version: ""
```

**Download `kubeconfig` file after the cluster is created**: use the fetch module to download the file using `inventory_dir` path as destination.
```yaml
kubernetes_fetch_kubeconfig: true
```

**Custom kubeadm configuration**: specify the configuration of component that kubeadm deploys.
This must be compliant with kubeadm configuration APIs.
By default, kubeadm setups a cluster with a single control-plane node.

```yaml
kubernetes_kubeadm_config:
  InitConfiguration:
    localAPIEndpoint:
      advertiseAddress: "{{ ansible_facts.default_ipv4.address }}"
  ClusterConfiguration:
    clusterName: "{{ kubernetes_cluster_name }}"
    networking:
      dnsDomain: "{{ kubernetes_dns_domain }}"
      podSubnet: "{{ kubernetes_pod_network }}"
      serviceSubnet: "{{ kubernetes_service_network }}"
  KubeletConfiguration:
    cgroupDriver: systemd
  KubeProxyConfiguration: {}
```

### High-Availability cluster
**Kubeadm certificate key**: specify the certificate key that will be used by kubeadm to join control-plane nodes.
**NOTE**: this variable is required only in High-Availability setups. It is recommended to store its value in a Vault.
```yaml
kubernetes_kubeadm_certificate_key: ""
```

**Custom kubeadm configuration**: specify the `loadbalancer_host` and `loadbalancer_port` variables of the external load balancer in front of control-plane nodes.

```yaml
kubernetes_kubeadm_config:
  InitConfiguration:
    certificateKey: "{{ kubernetes_kubeadm_certificate_key }}"
  ClusterConfiguration:
    controlPlaneEndpoint: "{{ loadbalancer_host }}:{{ loadbalancer_port }}"
    networking:
      dnsDomain: "{{ kubernetes_dns_domain }}"
      podSubnet: "{{ kubernetes_pod_network }}"
      serviceSubnet: "{{ kubernetes_service_network }}"
  KubeletConfiguration:
    cgroupDriver: systemd
  KubeProxyConfiguration: {}
```

### Optional configurations
**API Server configuration**: customize the API Server using Configuration APIs. 
```yaml
kubernetes_apiserver_config: {}
```


Dependencies
------------

**IMPORTANT**: This role does not specify any dependency to 3rd party roles.
However, a CNI plugin must be already installed on the host(s) in order to install Kubernetes using this role. So, you are free to choose which CNI plugin to install for your Kubernetes cluster.

This role has been tested together with [geerlingguy.containerd](https://github.com/geerlingguy/ansible-role-containerd) role, which installs `containerd` as default container runtime. `geerlingguy.containerd` role must run first in order to install Containerd before Kubernetes.

Required variables:
```yaml
# Set systemd as cgroup driver in config.toml (required for Kubernetes)
containerd_config_cgroup_driver_systemd: true
```

Optional variables:
```yaml
# Install latest version of containerd package (easy upgrades)
containerd_package_state: latest
```


Example Inventory
-----------------

```ini
# Super group of all kubernetes hosts (control-planes and workers)
kubernetes:
  children:
    k8s_example_cluster:

# Group for k8s_example cluster
k8s_example_cluster:
  children:
    # control-plane nodes
    k8s_example_masters:
      hosts:
        master1.example.org:
      vars:
        # mark the host(s) in this group as 'control-plane'
        kubernetes_role: control_plane
    # worker nodes
    k8s_example_workers:
      hosts:
        worker1.example.org:
      vars:
        # mark the host(s) in this group as 'workers'
        kubernetes_role: worker
```


Example Playbooks
-----------------

### Install and upgrade packages
Install Kubernetes using playbook `kubernetes.yml`, the same is useful to upgrade packages to a new version.
The playbook uses Ansible role [geerlingguy.containerd](https://github.com/geerlingguy/ansible-role-containerd) to install Containerd before Kubernetes.

```yaml
# Usage:
#   ansible-playbook -l <cluster-group> kubernetes.yml
---
- name: Install and upgrade packages on a Kubernetes host
  hosts: kubernetes
  become: true

  roles:
    - role: geerlingguy.containerd
      vars:
        # Set systemd as cgroup driver in config.toml (required for Kubernetes)
        containerd_config_cgroup_driver_systemd: true
        # Optional: Install latest version of containerd package (easy upgrades)
        containerd_package_state: latest
    - mcaliandro.kubernetes
```

### Create a new cluster
1. **Create a new cluster** using playbook `kubernetes/create_cluster.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <first-node> kubernetes/create_cluster.yml
    ---
    - name: Create a new Kubernetes cluster
      hosts: kubernetes
      become: true

      tasks:
        - ansible.builtin.import_role:
            name: mcaliandro.kubernetes
            tasks_from: create_cluster.yml
    ```

2. **Join nodes to an existing cluster** using playbook `kubernetes/join_node.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <host/group> kubernetes/join_node.yml
    ---
    - name: Join new node to an existing Kubernetes cluster
      hosts: kubernetes
      become: true
      serial: 1

      tasks:
        - ansible.builtin.import_role:
            name: mcaliandro.kubernetes
            tasks_from: join_node.yml
    ```

### Cluster upgrades

1. First, specify the new Kubernetes version by setting the variable `kubernetes_version`.

    **NOTE**: Skipping minor versions when upgrading is unsupported. For more info, see [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/).

2. **Upgrade Containerd and Kubernetes on hosts** using playbook `kubernetes.yml` (see [Install and upgrade packages](#install-and-upgrade-packages)).

3. **Upgrade a cluster** using playbook `kubernetes/upgrade_cluster.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <first-node> kubernetes/upgrade_cluster.yml
    ---
    - name: Upgrade a Kubernetes cluster using kubeadm
      hosts: kubernetes
      become: true

      tasks:
        - ansible.builtin.import_role:
            name: mcaliandro.kubernetes
            tasks_from: upgrade_cluster.yml
    ```

4. **Upgrade nodes of a cluster** using playbook `kubernetes/upgrade_nodes.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <host/group> kubernetes/upgrade_node.yml
    ---
    - name: Upgrade Kubernetes cluster nodes using kubeadm
      hosts: kubernetes
      become: true
      serial: 1

      tasks:
        - ansible.builtin.import_role:
            name: mcaliandro.kubernetes
            tasks_from: upgrade_node.yml
    ```


Compatibility
-------------

This role has been tested on these Linux distributions:
- Ubuntu Server LTS 22.04
- Ubuntu Server LTS 24.04
- Ubuntu Server LTS 26.04


License
-------

Apache License

Author Information
------------------

Created in 2026 by Matteo Caliandro, <mcaliandro.dev@gmail.com>