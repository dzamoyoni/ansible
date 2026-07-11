# Kubernetes Infrastructure & Lifecycle Automation with Ansible

This repository contains a production-grade suite of Ansible playbooks designed to manage the entire lifecycle of Kubernetes clusters. From initial multi-master high-availability bootstrapping to granular resource deployments, network troubleshooting, cluster backups, and absolute environment decommissioning, these playbooks automate day-1 and day-2 operations end-to-end.

---

## 📌 Repository Overview & Exact Scenarios

The playbooks in this repository are categorized into four lifecycle areas. Below is the precise operational scenario and backend behavior for every file:

### 1. Cluster Provisioning & Architecture

#### `stacked-ha-k8s.yaml`
* **Exact Scenario:** When you need to provision a **production-ready, multi-master Highly Available (HA) Kubernetes cluster** from scratch using a stacked etcd topology[cite: 1].
* **Backend Behavior:** Orchestrates a robust 9-phase rollout[cite: 1]. It passes pre-flight system checks, provisions Docker and `cri-dockerd`, boots up the primary control node, and generates dynamic configurations for **HAProxy** and **Keepalived**[cite: 1]. It deploys these load balancers as static pods fronted by a Virtual IP (VIP), installs the Calico CNI plugin, untaints master nodes, and sequentially scales out additional control planes and worker nodes[cite: 1].

#### `master-worker-k8s.yaml`
* **Exact Scenario:** When you need a **lightweight, non-HA development, testing, or staging cluster** featuring a single control plane node and multiple workers[cite: 3].
* **Backend Behavior:** Bypasses the infrastructure footprint of external load balancers, Keepalived, and dedicated VIPs[cite: 1, 3]. It handles cross-distribution OS configuration (Ubuntu, Debian, RHEL, CentOS, Rocky Linux), permanently disables system swap, installs the Docker/containerd runtime via `cri-dockerd`, configures the master using its standard host IP, installs Calico, and joins your worker nodes[cite: 3].

### 2. Day-2 Management & Resource Deployment

#### `add-kubeconfig-local.yaml`
* **Exact Scenario:** Right after initializing a new cluster, when you want to **securely configure access from your local laptop/admin machine** without overwriting, wiping out, or disrupting your existing `~/.kube/config` profiles.
* **Backend Behavior:** Connects to the primary control plane, slurps the raw `admin.conf` via Ansible, and uses an integrated local Python script to intercept and rewrite the cluster, user, and context metadata blocks to a name you specify at a terminal prompt[cite: 10]. It merges these components into your local config, sets it as current, and runs a validation test (`kubectl get nodes`)[cite: 10]. If connectivity fails due to firewalls or routing, it automatically rolls back the entire merge to preserve file integrity[cite: 10].

#### `k8s-apps-deployments.yaml`
* **Exact Scenario:** When you have an active cluster and need to **deploy a full application stack** (e.g., a multi-tier QA environment) directly from YAML manifests residing on your local administration machine[cite: 6].
* **Backend Behavior:** Connects to the remote control plane, installs host-level dependencies (`python3-pip`, `python3-kubernetes`), creates the target application namespace, and uses local fileglob lookups to apply your local manifests in a strict, logical sequence: **Secrets ➡️ ConfigMaps ➡️ Deployments ➡️ Services ➡️ StatefulSets**[cite: 6]. It uses an intentional block-level `ignore_errors: true` toggle to ensure that a single failing manifest does not catastrophically abort the entire application rollout block[cite: 6].

### 3. Backup, Restore, & Cluster Diagnostics

#### `k8s_backup.yml`
* **Exact Scenario:** When implementing a **GitOps pipeline or routine backup schedule** to capture the exact, stateful blueprint of running workloads across custom namespaces[cite: 5].
* **Backend Behavior:** Detects available control plane cluster credentials, excludes standard platform domains (`kube-system`, `kube-public`, `kube-node-lease`), and sweeps selected API objects[cite: 5]. It checks for host installations of `yq` or `kubectl-neat` to intelligently strip out transient, auto-generated platform metadata (such as `creationTimestamp`, `uid`, `resourceVersion`, and `status` fields)[cite: 5]. It aggregates these clean YAML blueprints into a remote archive, pulls the tarball locally to your controller, extracts it into a structured directory tree, and scrubs the remote host footprint[cite: 5].

#### `k8s_restore.yml`
* **Exact Scenario:** When recovering from a **critical namespace-level failure** or replicating an isolated application workspace layout into a completely blank cluster[cite: 4, 5].
* **Backend Behavior:** Systematically reads through your extracted backup directory, builds missing namespaces on the target cluster (safely avoiding platform namespace collisions), and passes the files through an inline `yq` filter to strip any residual cluster footprints (like `volumeName`, `clusterIP`, or `status` parameters) before piping them directly into `kubectl apply`[cite: 4].

#### `diagnose-network.yaml`
* **Exact Scenario:** When nodes are dropping to a `NotReady` state, pods cannot establish cross-node communications, or you suspect **underlying network blockages/firewall constraints** in an HA setup[cite: 7].
* **Backend Behavior:** Validates listening states on all core cluster ports (such as `6443`, `10250`, `2379`, and `2380`), maps host routing tables, checks active status of host firewalls (`ufw` or `firewalld`), and runs a comprehensive mesh connectivity matrix checking `ping`, `SSH`, `kubelet`, and `etcd` peer-to-peer traffic between control elements[cite: 7]. It also prints out the active `cali-INPUT` and `cali-FORWARD` iptables rule structures to catch breaking Calico network policies[cite: 7].

#### `debug-kubelet.yaml`
* **Exact Scenario:** When a single node is showing a **persistent `NotReady` status**, or the underlying host `kubelet` service is actively crashing and failing to talk to the container layer[cite: 9].
* **Backend Behavior:** Directly targets the failing node to inspect `/etc/default/kubelet` arguments, reads active service properties, and pulls down the last 50 lines of system logs out of `journalctl`[cite: 9]. It goes deep into the host environment to check the runtime status of `cri-docker` and `containerd`, confirms whether host swap has accidentally been re-enabled, and verifies that the `br_netfilter` and `overlay` kernel modules remain successfully loaded[cite: 9].

### 4. Teardown & Decommissioning

#### `reset-cluster.yaml`
* **Exact Scenario:** When a cluster provisioning run fails mid-way, or you want to **cleanly un-join a node or reset the cluster core** back to a blank slate without deleting the docker runtime package itself[cite: 2].
* **Backend Behavior:** Safely drains and deletes the target node out of the API (if reachable), executes a forced `kubeadm reset`, terminates the `kubelet` daemon, and recursively deletes all residual directories (`/etc/kubernetes`, `/var/lib/etcd`, CNI binaries, etc.)[cite: 2]. It completely flushes all iptables/ipvs rule tables generated by `kube-proxy`/`calico`, drops virtual network adapters (`cni0`, `cali*`), and bounces the container runtime daemons to clear out zombie tasks[cite: 2].

#### `destroy-docker-k8s.yaml`
* **Exact Scenario:** When you want to **completely decommission an entire infrastructure fleet**, wipe out every trace of the environment, and return all servers to a clean, base OS state.
* **Backend Behavior:** Serves as a total infrastructure eraser[cite: 8]. It triggers a prominent terminal warning banner detailing the extent of the data loss and halts execution until you manually type the phrase `destroy`[cite: 8]. Once verified, it executes a clean cluster reset and then **purges and uninstalls all Kubernetes, Docker, containerd, HAProxy, and Keepalived application packages**[cite: 8]. It forces the destruction of all local container images, persistent volumes, host kernel overrides, network interfaces, routing configurations, and cached temp files[cite: 8].

---

## 🛠️ Infrastructure Requirements

Before running the provisioning playbooks, ensure your target hosts meet these baseline parameters:

* **Hardware Footprint:** Minimum `2 GB RAM` (or `1800 MB+` of strictly free memory available) per node[cite: 3].
* **Runtime Layer:** Configured out-of-the-box to track through Docker backed by the `cri-dockerd` engine layer[cite: 1, 3].
* **Inventory Structure:** High-availability playbooks require your target nodes to be explicitly split into the `k8s_control_plane` and `k8s_workers` inventory host groups[cite: 1, 3].
* **Global Variables:** Variables such as `k8s_vip`, `kube_api_bind_port`, and `haproxy_frontend_port` must be defined within your group variables or inventory file prior to deploying an HA architecture[cite: 1].

---

## 🚀 Execution Guide

### 1. Bootstrapping a New HA Cluster
To run the full end-to-end multi-phase deployment for a High Availability cluster:
```bash
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml
or 
ansible-playbook -i inventory/hosts.yaml <playbook_name>.yaml


### 2. Micro-Targeting Deployment Stages (Tags)

The provisioning architecture features built-in tag targets, allowing you to run specific steps without executing the full playbook:

# Run only pre-flight environment validations
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml --tags "validation"

# Target only the installation/verification of the container runtime (Docker/cri-dockerd)
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml --tags "docker"

# Deploy only Keepalived & HAProxy static control plane load balancers
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml --tags "ha"

# Scale out and join only the designated worker nodes to the active cluster
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml --tags "join-workers"

3. Merging Remote Contexts Locally
To securely capture cluster credentials and map them locally:

ansible-playbook -i inventory/hosts.yaml add-kubeconfig-local.yaml
or 
ansible-playbook -i inventory/ha-hosts.yaml stacked-ha-k8s.yaml

Note: You will be prompted directly in your terminal to provide an alphanumeric name for your cluster to keep your context configurations isolated[cite: 10].

4. Executing Smart Workload Backups
  To query your active namespaces, scrub off runtime metadata parameters, and sync a structured GitOps backup tree locally[cite: 5]:
    ``` ansible-playbook -i inventory/hosts.yaml k8s_backup.yml```
    Tip: The resulting archive directory matches the naming convention: ../k8s-backups/<ansible_host>-<date>[cite: 5].

5. Running Full Disaster Recovery Restores
    To recreate application workspaces from a clean backup snapshot folder while skipping default platform layers:
      ``` ansible-playbook -i inventory/hosts.yaml k8s_restore.yml -e "backup_root=k8s-backup-2026-01-20" ```

6. Full Infrastructure Destruction
    To purge all components, packages, runtime engines, and networking configurations:
    ansible-playbook -i inventory/hosts.yaml destroy-docker-k8s.yaml
    Note: This playbook is highly destructive and requires typing the confirmation string destroy to proceed past the safety gate[cite: 8].