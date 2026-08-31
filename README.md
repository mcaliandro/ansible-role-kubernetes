Ansible role for Kubernetes
=========

Install and configure on-premises Kubernetes clusters.

This role is designed to bootstrap and manage Kubernetes clusters using `kubeadm` method, as well as supporting the customization of Kubernetes components with kubeadm APIs.

Requirements
------------

Ansible collections:
- [ansible.posix](https://galaxy.ansible.com/ui/repo/published/ansible/posix/)
- [community.general](https://galaxy.ansible.com/ui/repo/published/community/general/)

**IMPORTANT**: This role does not install any Container Runtime. It has been tested with Ansible role [geerlingguy.containerd](https://github.com/geerlingguy/ansible-role-containerd). However, you are free to choose which one fits your need. 


Role Variables
--------------

**NOTE**: In case you choose `geerlingguy.containerd` for CRI installation, then make sure to specify also the following variables:

```yaml
# Set systemd as cgroup driver in config.toml (required for Kubernetes)
containerd_config_cgroup_driver_systemd: true
# Install latest version of containerd package (easy upgrades)
containerd_package_state: latest
```

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
**NOTE**: this variable is required when `kubernetes_cni_plugin == "calico"`.
```yaml
kubernetes_cni_plugin_version: ""
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

None.


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
        kubernetes_role: control_plane
    # worker nodes
    k8s_example_workers:
      hosts:
        worker1.example.org:
      vars:
        kubernetes_role: worker
```


Example Playbooks
-----------------

### Basic playbook

Install Kubernetes using this playbook.

```yaml
# Usage:
#   ansible-playbook -l <cluster-group> kubernetes.yml
---
- name: Install Kubernetes
  hosts: kubernetes
  become: true

  roles:
    - mcaliandro.kubernetes
```


### First install
1. **Install Containerd and Kubernetes on hosts** using playbook `kubernetes/install.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <cluster-group> kubernetes/install.yml
    ---
    - name: Install and configure a Kubernetes host
      hosts: kubernetes
      become: true

      roles:
        - geerlingguy.containerd
        - mcaliandro.kubernetes
    ```

2. **Create a new cluster** using playbook `kubernetes/create_cluster.yml`.

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

3. **Join nodes to an existing cluster** using playbook `kubernetes/join_node.yml`.

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

2. **Upgrade Containerd and Kubernetes on hosts** using playbook `kubernetes/install.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <cluster-group> playbooks/kubernetes.yml
    ---
    - name: Install and configure a Kubernetes host
      hosts: kubernetes
      become: true

      roles:
        - geerlingguy.containerd
        - mcaliandro.kubernetes
    ```

3. **Upgrade a cluster** using playbook `kubernetes/upgrade_cluster.yml`.

    ```yaml
    # Usage:
    #   ansible-playbook -l <first-node> kubernetes/upgrade_cluster.yml
    ---
    - name: Upgrade a Kubernetes cluster
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
    - name: Upgrade Kubernetes cluster nodes
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