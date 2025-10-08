# Enhanced Ansible Kubernetes Setup

This repository provides a generic and best-practice enhanced Ansible setup for deploying a Kubernetes cluster. It replaces the original Flannel CNI with Cilium and includes fixes for the containerd `systemdCgroup` configuration.

## Table of Contents
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Inventory Setup](#inventory-setup)
- [Usage](#usage)
- [Roles Overview](#roles-overview)
- [Customization](#customization)
- [Cilium CNI](#cilium-cni)
- [Containerd `systemdCgroup` Fix](#containerd-systemdcgroup-fix)

## Project Structure

```
ansible-k8s-enhanced/
├── ansible.cfg
├── inventory/
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── lb.yml
│   └── hosts.yml
├── playbooks/
│   ├── bootstrap-k8s-cluster.yml
│   ├── reset-kubeadm.yml
│   ├── setup-commontools.yml
│   ├── setup-containerd.yml
│   ├── setup-docker.yml
│   ├── setup-k8s-lb.yml
│   ├── setup-k8s.yml
│   ├── setup-ports.yml
│   └── setup-selinux.yml
└── roles/
    ├── cilium/
    │   └── tasks/
    │       └── main.yml
    ├── common-tools/
    │   └── tasks/
    │       └── main.yml
    ├── containerd/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       └── config.toml.j2
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    ├── haproxy/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       └── haproxy.cfg.j2
    ├── k8s-presetup/
    │   ├── defaults/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    ├── keepalived/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       └── keepalived.conf.j2
    ├── kubernetes/
    │   ├── tasks/
    │   │   └── master.yml
    │   └── templates/
    │       └── kubeadm-config.yaml.j2
    ├── ports/
    │   └── tasks/
    │       └── main.yml
    └── selinux/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── tasks/
            └── main.yml
```

## Prerequisites

### Control Node
- **Ansible**: Version 2.9 or higher is recommended.
- **Python**: Python 3 installed.
- **`kubernetes.core` collection**: Install with `ansible-galaxy collection install kubernetes.core`.

### Target Nodes (Kubernetes Nodes)
- **Operating System**: CentOS 7/8, Ubuntu 18.04/20.04, or RHEL 7/8.
- **Sudo Privileges**: A user with `sudo` access and passwordless `sudo` configured for the Ansible user.
- **Internet Access**: Required for downloading packages and container images.
- **Minimum Resources**: At least 2 vCPUs and 2GB RAM for master nodes, 1 vCPU and 2GB RAM for worker nodes.

## Inventory Setup

Configure your target hosts in `inventory/hosts.yml`.

Example `inventory/hosts.yml`:

```yaml
all:
  hosts:
    master1:
      ansible_host: <MASTER_NODE_IP_1>
    master2:
      ansible_host: <MASTER_NODE_IP_2>
    worker1:
      ansible_host: <WORKER_NODE_IP_1>
    worker2:
      ansible_host: <WORKER_NODE_IP_2>
  children:
    masters:
      hosts:
        master1:
        master2:
    workers:
      hosts:
        worker1:
        worker2:
    lb:
      hosts:
        master1:
        master2:
```

Adjust `group_vars/all.yml` and `group_vars/lb.yml` for global variables and load balancer specific configurations, respectively.

## Usage

1.  **Prepare your inventory**: Edit `inventory/hosts.yml` with your server details.
2.  **Configure variables**: Adjust variables in `inventory/group_vars/all.yml` and other role `defaults/main.yml` files as needed.
3.  **Run the playbooks**:

    - **Bootstrap Kubernetes Cluster**: This is the main playbook to set up the entire cluster.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/bootstrap-k8s-cluster.yml
      ```

    - **Reset Kubeadm**: Use this to reset a Kubernetes node.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/reset-kubeadm.yml
      ```

    - **Setup Common Tools**: Installs basic utilities.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/setup-commontools.yml
      ```

    - **Setup Containerd**: Installs and configures Containerd.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/setup-containerd.yml
      ```

    - **Setup SELinux**: Configures SELinux for Kubernetes.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/setup-selinux.yml
      ```

    - **Setup Ports**: Configures firewall rules.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/setup-ports.yml
      ```

    - **Setup K8s Load Balancer**: Configures HAProxy and Keepalived for control plane high availability.
      ```bash
      ansible-playbook -i inventory/hosts.yml playbooks/setup-k8s-lb.yml
      ```

## Roles Overview

-   **`cilium`**: Installs and configures Cilium as the Container Network Interface (CNI).
-   **`common-tools`**: Installs essential packages like `git`, `wget`, `curl`, etc.
-   **`containerd`**: Installs and configures the Containerd runtime.
-   **`docker`**: (Optional) Installs Docker. Kubernetes now primarily uses Containerd.
-   **`haproxy`**: Configures HAProxy for load balancing Kubernetes API servers.
-   **`k8s-presetup`**: Performs pre-installation tasks for Kubernetes, such as disabling swap, configuring kernel modules, and setting up `sysctl` parameters.
-   **`keepalived`**: Configures Keepalived for virtual IP (VIP) management for high availability.
-   **`kubernetes`**: Installs `kubeadm`, `kubelet`, `kubectl`, and initializes/joins Kubernetes cluster nodes.
-   **`ports`**: Manages firewall rules to open necessary ports for Kubernetes.
-   **`selinux`**: Configures SELinux policies required for Kubernetes.

## Customization

Most variables are defined in `inventory/group_vars/all.yml` or within the `defaults/main.yml` of each role. Review these files to customize:

-   `kubernetes_version`: The desired Kubernetes version.
-   `pod_network_cidr`: The CIDR for the Pod network (e.g., `10.244.0.0/16`).
-   `service_network_cidr`: The CIDR for the Service network (e.g., `10.96.0.0/12`).
-   `containerd_version`: The desired Containerd version.
-   Load balancer IPs and ports.

## Cilium CNI

This setup uses **Cilium** as the CNI plugin for Kubernetes. The installation is handled automatically by the `cilium` role within the `setup-k8s.yml` playbook.

## Containerd `systemdCgroup` Fix

The `containerd` role has been updated to correctly configure the `SystemdCgroup = true` setting within the `config.toml` file. This ensures that Containerd properly integrates with `systemd` for cgroup management, which is a best practice for Kubernetes environments.



## Recommended Playbook Execution Order

For a complete Kubernetes cluster setup, it is recommended to run the playbooks in the following order:

1.  **Setup Common Tools**: Installs essential packages on all nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-commontools.yml
    ```

2.  **Setup SELinux**: Configures SELinux policies on all nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-selinux.yml
    ```

3.  **Setup Ports**: Configures firewall rules on all nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-ports.yml
    ```

4.  **Setup Containerd**: Installs and configures Containerd runtime on all nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-containerd.yml
    ```

5.  **Setup K8s Load Balancer**: Configures HAProxy and Keepalived for control plane high availability (run on designated load balancer nodes, typically master nodes).
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-k8s-lb.yml
    ```

6.  **Setup Kubernetes Components**: Installs `kubeadm`, `kubelet`, `kubectl`, and deploys Cilium CNI on all nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/setup-k8s.yml
    ```

7.  **Bootstrap Kubernetes Cluster**: Initializes the Kubernetes control plane and joins worker nodes.
    ```bash
    ansible-playbook -i inventory/hosts.yml playbooks/bootstrap-k8s-cluster.yml
    ```

**Note**: The `reset-kubeadm.yml` playbook is for resetting a Kubernetes node and should only be used when you need to clean up a node before rejoining or reinitializing.
