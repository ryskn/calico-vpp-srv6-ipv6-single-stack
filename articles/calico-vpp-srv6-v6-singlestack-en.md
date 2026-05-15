---
title: "Running Calico-VPP + SRv6 with IPv6 Single-Stack"
emoji: "🛰️"
type: "tech"
topics: ["kubernetes", "calico", "vpp", "srv6", "ipv6"]
published: true
---

This article describes how to set up the Calico-VPP SRv6 dataplane in an IPv6 single-stack configuration on a three-node VM environment on Proxmox (one master, two workers).

Because SRv6 assumes an IPv6 underlay, it is more natural to align Pods and Services to v6 as well. When I previously built this as dual-stack, IPv4 Services through cnat did not work and the agent crashed with cnat ENOENT errors. The v4 path was not stable enough for practical use, so I consolidated to a single family.

:::message
**What works with v3.32.0:** The procedure described here was run with Calico-VPP **v3.32.0** (= the official projectcalico/vpp-dataplane release). It works with the official images (`docker.io/calicovpp/{vpp,agent}:v3.32.0`) and **does not require a fork build**. Up through earlier 3.31.x releases, there were three SRv6 install-related bugs ([PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)) and a cnat ENOENT issue where the agent crashed in v6 single-stack ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982) + [VPP Gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604)), so a custom image build was required. In v3.32.0, all of these have been merged.
:::

## Configuration

| Component | IPv4 (mgmt) | IPv6 ULA (k8s control-plane) |
|---|---|---|
| **infra01** (DNS / dnf-proxy) | 192.168.1.6 | (auto SLAAC) |
| **master** | 192.168.1.10 | fd00:1::10 |
| **worker-1** | 192.168.1.11 | fd00:1::11 |
| **worker-2** | 192.168.1.12 | fd00:1::12 |

- Management NW: `192.168.1.0/24` (IPv4)
- IPv6 underlay: `240b:…/64` SLAAC via RA from the Proxmox host + static ULA (`fd00:1::/64`) added to eth0
- Internal DNS: **NSD (authoritative for `ryskn.k8s`)** + **unbound (recursive resolver)** on infra01
- Local dnf proxy: nginx on infra01 (`http://dnf-proxy.ryskn.k8s/almalinux/...`)
- Pod CIDR: `fd20::/64` (explicitly specified as an IPPool in the Installation CR)
- Service CIDR: `fd30::/108`
- Pod allocation: **LocalSID pools under `fcff:0:0:<per-node>AA::/64`** (Pod IP = End.DT6 SID)

## Why IPv6 Single-Stack?

In IPv6 single-stack, cnat's IPv4 path has an empty `INCLUDE_V4`, and when the agent reconcile loop sends a disable operation, the agent crashes with ENOENT ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982)). Even in dual-stack, cnat's IPv4 NAT is unstable, and I have encountered cases where Service connectivity intermittently fails. Since SRv6 assumes a v6 underlay, aligning Pods, Services, and the control plane to v6 reduces the number of trip-wires.

## 0. infra (DNS / dnf-proxy) Prerequisites

Without an internal DNS server that serves the `ryskn.k8s` zone, names such as `master-1.ryskn.k8s` and `dnf-proxy.ryskn.k8s` cannot be resolved, and the setup cannot proceed.

On infra01 (= 192.168.1.6):
- Use **NSD** to serve the `ryskn.k8s` zone authoritatively (example: `dnf-proxy IN A 192.168.1.6`)
- Use **unbound** as the recursive resolver, forwarding only `ryskn.k8s` to `127.0.0.1@1053` (= NSD) as a stub-zone, and forwarding everything else to `1.1.1.1`
- Use nginx to proxy `/almalinux/`, `/cri-o/`, and so on from the official AlmaLinux mirrors (= local dnf-proxy)

Make sure both services are enabled with `systemctl enable --now`. It is **easy to forget auto-start after boot**, and after a reboot everything breaks if DNS is missing.

## 1. Create Proxmox VMs

On the Proxmox host, clone from the AlmaLinux 10 cloud-init template (VMID 9001):

```bash
# master (VMID 103)
qm clone 9001 103 --name master --full
qm set 103 --cores 6 --sockets 1 --cpu host --memory 4096 --balloon 0 \
    --net0 virtio,bridge=vmbr0 --agent enabled=1 --onboot 1
qm resize 103 scsi0 50G
qm set 103 --ipconfig0 ip=192.168.1.10/24,gw=192.168.1.1,ip6=auto
qm set 103 --nameserver 192.168.1.6 --searchdomain ryskn.k8s

# worker-1 (VMID 101)
qm clone 9001 101 --name worker-1 --full
qm set 101 --cores 6 --sockets 1 --cpu host --memory 6144 --balloon 0 \
    --net0 virtio,bridge=vmbr0 --agent enabled=1 --onboot 1
qm resize 101 scsi0 50G
qm set 101 --ipconfig0 ip=192.168.1.11/24,gw=192.168.1.1,ip6=auto
qm set 101 --nameserver 192.168.1.6 --searchdomain ryskn.k8s

# worker-2 (VMID 102) is set up the same way (ip=192.168.1.12)

qm start 103 && qm start 101 && qm start 102
```

## 2. Disable Multicast Snooping on Proxmox vmbr0

This is **one of the biggest pitfalls**. In my test environment (Proxmox VE 8.x), vmbr0 had `multicast_snooping=1` and no multicast querier was running. In that state, IPv6 Neighbor Solicitations (`ff02::1:ff…`) are not flooded between VMs, so ND cannot resolve for the SRv6-encapsulated packets (I confirmed that `show ip6 neighbors` inside the VPP container stayed empty). As a result, v6 traffic could not reach anything outside the VM.

```bash
# Check (default is 1)
cat /sys/class/net/vmbr0/bridge/multicast_snooping

# Apply immediately
echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping

# Persist: add to the vmbr0 block in /etc/network/interfaces
#   post-up echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping
```

## 3. AlmaLinux cloud-init Traps (Three of Them)

These were **the most painful issues this time**. When using an AlmaLinux cloud-init template, the following are reset on every reboot:

### 3-1. `/etc/hosts` Is Regenerated by `manage_etc_hosts: True`

Manual entries such as `fd00:1::10 master-1.ryskn.k8s` disappear on every boot.

**Fix**: Set `manage_etc_hosts: false` in `/etc/cloud/cloud.cfg`.

### 3-2. `/etc/sysconfig/kubelet` Is Reset to Empty

`KUBELET_EXTRA_ARGS=--node-ip=fd00:1::10 ...` is rewritten to empty, and kubelet re-registers with the IPv4 InternalIP (= contradicting the cluster's v6 single-stack design).

**Fix**: Make it immutable with `chattr +i /etc/sysconfig/kubelet`.

### 3-3. NetworkManager Regenerates `/etc/resolv.conf`

Even if you write `nameserver 192.168.1.6` (internal DNS), it disappears. As a result, `dnf-proxy.ryskn.k8s` cannot be resolved, and every install fails.

**Fix**: Make it immutable with `chattr +i /etc/resolv.conf`.

## 4. Common Preparation on All Nodes

SSH as root to each node (master + two workers) (username `ryosuke`, then run `sudo bash`):

```bash
# === [0/6] kubernetes / cri-o repos (official pkgs.k8s.io) ===
# kubernetes 1.36 / cri-o 1.32 stable. cri-o 1.33+ has no stable release yet, so we pin to 1.32
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
EOF
cat > /etc/yum.repos.d/cri-o.repo <<EOF
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF
dnf clean all && dnf makecache

# === [0.5/6] DNS (locked with chattr +i) ===
chattr -i /etc/resolv.conf 2>/dev/null || true
printf "nameserver 192.168.1.6\nsearch ryskn.k8s\n" > /etc/resolv.conf
chattr +i /etc/resolv.conf

# === [0.6/6] Add ULA IPv6 to eth0 ===
# In IPv6 single-stack k8s, the apiserver must bind to a v6 IP, and the
# Proxmox SLAAC v6 address (240b::/64) is taken by VPP, so add a static ULA to each node.
HOST_LAST_OCTET=$(ip -4 -o addr show eth0 | awk '{print $4}' | cut -d. -f4 | cut -d/ -f1)
ULA_ADDR="fd00:1::${HOST_LAST_OCTET}/64"
NM_CONN=$(nmcli -g NAME,DEVICE con show --active | awk -F: '$2=="eth0"{print $1; exit}')
nmcli con mod "$NM_CONN" +ipv6.addresses "$ULA_ADDR"
nmcli con mod "$NM_CONN" ipv6.method auto
nmcli con up "$NM_CONN"

# === [0.7/6] /etc/hosts (disable cloud-init interference) ===
sed -i 's/^manage_etc_hosts:.*/manage_etc_hosts: false/' /etc/cloud/cloud.cfg
grep -q '^manage_etc_hosts:' /etc/cloud/cloud.cfg || \
    echo 'manage_etc_hosts: false' >> /etc/cloud/cloud.cfg
sed -i '/ryskn.k8s$/d' /etc/hosts
cat >> /etc/hosts <<EOF
fd00:1::10 master-1.ryskn.k8s
fd00:1::11 worker-1.ryskn.k8s
fd00:1::12 worker-2.ryskn.k8s
EOF

# === [0.8/6] Dummy route for the v6 Service CIDR ===
# Workaround so that locally-originated packets to the Service CIDR are not dropped as unreachable
ip -6 route replace fd30::/108 dev lo
cat > /etc/systemd/system/k8s-v6-service-route.service <<EOF
[Unit]
Description=Add dummy v6 route for k8s Service CIDR
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
ExecStart=/sbin/ip -6 route replace fd30::/108 dev lo
RemainAfterExit=true
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now k8s-v6-service-route.service

# === [1/6] swap off ===
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab

# === [2/6] SELinux permissive ===
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# === [3/6] kernel modules ===
dnf install -y kernel-modules-extra
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
cat > /etc/modules-load.d/vpp.conf <<EOF
vfio-pci
uio_pci_generic
EOF
modprobe overlay br_netfilter vfio-pci uio_pci_generic

# === [4/6] sysctl + hugepages ===
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.forwarding        = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.conf.default.rp_filter     = 0
net.ipv4.conf.eth0.rp_filter        = 0
EOF
cat > /etc/sysctl.d/hugepages.conf <<EOF
vm.nr_hugepages = 512
EOF
sysctl --system

# === [5/6] cri-o ===
dnf install -y cri-o
systemctl enable --now crio

# === [6/6] kubeadm / kubelet / kubectl + prevent cloud-init interference ===
dnf install -y kubeadm kubelet kubectl

NODE_ULA="fd00:1::${HOST_LAST_OCTET}"
chattr -i /etc/sysconfig/kubelet 2>/dev/null || true
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///var/run/crio/crio.sock --node-ip=${NODE_ULA}
EOF
chattr +i /etc/sysconfig/kubelet  # prevent it from being rewritten to empty after reboot

systemctl daemon-reload && systemctl enable kubelet
```

:::message
**Write `KUBELET_EXTRA_ARGS` in `/etc/sysconfig/kubelet`**. kubeadm's standard drop-in (`/usr/lib/.../10-kubeadm.conf`) includes `EnvironmentFile=-/etc/sysconfig/kubelet`, so the `KUBELET_EXTRA_ARGS` written there is what takes effect. In my environment, putting the same variable in a separate drop-in (`20-crio.conf`) under `Environment=` did not take effect, so writing it to `/etc/sysconfig/kubelet` is the reliable path.
:::

## 5. Initialize the Master

On master (192.168.1.10):

```bash
sudo kubeadm init \
    --cri-socket=unix:///var/run/crio/crio.sock \
    --apiserver-advertise-address=fd00:1::10 \
    --pod-network-cidr=fd20::/64 \
    --service-cidr=fd30::/108 \
    --control-plane-endpoint=master-1.ryskn.k8s

# kubeconfig (for both root and ryosuke)
mkdir -p /root/.kube && cp -f /etc/kubernetes/admin.conf /root/.kube/config
mkdir -p /home/ryosuke/.kube && cp -f /etc/kubernetes/admin.conf /home/ryosuke/.kube/config
chown -R ryosuke:ryosuke /home/ryosuke/.kube

sudo kubeadm token create --print-join-command > /home/ryosuke/k8s-join.sh
chown ryosuke:ryosuke /home/ryosuke/k8s-join.sh
```

## 6. Join Workers

On each worker:

```bash
JOIN_CMD=$(ssh ryosuke@192.168.1.10 'cat ~/k8s-join.sh')
sudo bash -c "$JOIN_CMD --cri-socket=unix:///var/run/crio/crio.sock"
```

Check:

```bash
kubectl get nodes -o wide
# NAME                 STATUS     INTERNAL-IP   VERSION
# master.ryskn.k8s     Ready      fd00:1::10    v1.36.0
# worker-1.ryskn.k8s   NotReady   fd00:1::11    v1.36.0
# worker-2.ryskn.k8s   NotReady   fd00:1::12    v1.36.0
```

`NotReady` is expected because the CNI has not been installed yet (= the next section fixes it). If InternalIP is **v6**, that proves `chattr +i /etc/sysconfig/kubelet` is working.

## 7. tigera-operator + Calico Installation (v3.32.0)

On master:

```bash
# operator-crds + tigera-operator (v3.32.0)
kubectl apply --server-side -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/operator-crds.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
kubectl -n tigera-operator rollout status deployment/tigera-operator --timeout=300s

until kubectl get crd installations.operator.tigera.io >/dev/null 2>&1; do sleep 5; done
```

Apply the Installation CR with **only v6 ipPools explicitly specified** (= prevents the operator from creating a v4 pool on its own):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    linuxDataplane: VPP
    ipPools:
    - cidr: fd20::/64
      name: default-ipv6-ippool
      blockSize: 122
      encapsulation: None
      nodeSelector: all()
      natOutgoing: Disabled
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

## 8. Calico-VPP Manifest (kustomize)

Create a custom overlay under the repository's `yaml/overlays/`. The base is `test-vagrant` (= the official generic base).

### 8-1. kustomization.yaml

```yaml
bases:
- ../test-vagrant
resources:
- srv6res-patch.yaml
- rbac-patch.yaml
patchesStrategicMerge:
- srv6-cm-patch.yaml
- config-patch.yaml
- image-patch.yaml
```

### 8-2. config-patch.yaml (VPP uplink)

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-vpp-config
  namespace: calico-vpp-dataplane
data:
  CALICOVPP_INITIAL_CONFIG: |-
    {
      "vppStartupSleepSeconds": 1,
      "corePattern": "/var/lib/vpp/vppcore.%e.%p",
      "defaultGWs": "192.168.1.1"
    }
  CALICOVPP_INTERFACES: |-
    {
      "uplinkInterfaces": [
        {
          "interfaceName": "eth0",
          "vppDriver": "virtio"
        }
      ]
    }
```

### 8-3. srv6-cm-patch.yaml (Enable SRv6 + Set the Service CIDR to v6)

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-vpp-config
  namespace: calico-vpp-dataplane
data:
  SERVICE_PREFIX: "fd30::/108"
  CALICOVPP_DEBUG: |-
    { "gsoEnabled": false }
  CALICOVPP_SRV6: |-
    {
      "policyPool": "cafe::/118",
      "localsidPool": "fcff::/48"
    }
  CALICOVPP_FEATURE_GATES: |-
    { "srv6Enabled": true }
```

:::message alert
**Do not leave `SERVICE_PREFIX` at the default (`10.96.0.0/12`)**. In v6 single-stack, the apiserver Service IP is `fd30::1`, but if `SERVICE_PREFIX` stays at the v4 default, Calico-VPP does not treat the v6 Service CIDR as a NAT target, and kubelet → apiserver connectivity fails. This cost me a full day on this cluster.
:::

### 8-4. srv6res-patch.yaml (LocalSID Pools)

The LocalSID pools must strictly match the node names (`sr-localsids-pool-<hostname>`). The Calico-VPP agent searches for pools using this naming convention.

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: default-ipv6-ippool
  spec:
    blockSize: 122
    cidr: fd20::/64
    ipipMode: Never
    nodeSelector: all()
    vxlanMode: Never
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: sr-policies-pool
  spec:
    blockSize: 122
    cidr: cafe::/118
    ipipMode: Never
    nodeSelector: '!all()'
    vxlanMode: Never
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: sr-localsids-pool-master.ryskn.k8s
  spec:
    cidr: fcff:0:0:00AA::/64
    ipipMode: Never
    nodeSelector: kubernetes.io/hostname == 'master.ryskn.k8s'
    vxlanMode: Never
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: sr-localsids-pool-worker-1.ryskn.k8s
  spec:
    cidr: fcff:0:0:11AA::/64
    ipipMode: Never
    nodeSelector: kubernetes.io/hostname == 'worker-1.ryskn.k8s'
    vxlanMode: Never
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: sr-localsids-pool-worker-2.ryskn.k8s
  spec:
    cidr: fcff:0:0:12AA::/64
    ipipMode: Never
    nodeSelector: kubernetes.io/hostname == 'worker-2.ryskn.k8s'
    vxlanMode: Never
```

### 8-5. rbac-patch.yaml (ipreservations Permissions)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: calico-vpp-node-role-srv6-extra
rules:
- apiGroups: ["crd.projectcalico.org"]
  resources: ["ipreservations"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-vpp-node-srv6-extra
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-vpp-node-role-srv6-extra
subjects:
- kind: ServiceAccount
  name: calico-vpp-node-sa
  namespace: calico-vpp-dataplane
```

### 8-6. image-patch.yaml (Specify the Official v3.32.0 Images)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-vpp-node
  namespace: calico-vpp-dataplane
spec:
  template:
    spec:
      containers:
      - name: vpp
        image: docker.io/calicovpp/vpp:v3.32.0
        imagePullPolicy: IfNotPresent
      - name: agent
        image: docker.io/calicovpp/agent:v3.32.0
        imagePullPolicy: IfNotPresent
```

:::message
This works with the **official v3.32.0 images**. Previously, a fork build was required because of three SRv6 install-related bugs ([PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)) and cnat ENOENT ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982) + [VPP Gerrit #45604](https://gerrit.fd.io/r/c/vpp/+/45604)), but all of them have been merged in v3.32.0.
:::

### 8-7. Generate and Apply

```bash
kubectl kustomize yaml/overlays/dev-ryskn-srv6 > calico-vpp-srv6.yaml
kubectl apply -f calico-vpp-srv6.yaml
```

## 9. Verification

```bash
# Nodes Ready
kubectl get nodes -o wide
# NAME                 STATUS  INTERNAL-IP
# master.ryskn.k8s     Ready   fd00:1::10
# worker-1.ryskn.k8s   Ready   fd00:1::11
# worker-2.ryskn.k8s   Ready   fd00:1::12

# Calico-VPP DaemonSet is 2/2 Running, image is v3.32.0
kubectl -n calico-vpp-dataplane get ds calico-vpp-node -o wide

# Pod IPs are allocated from the SRv6 LocalSID pools (fcff:0:0:<node>AA::/64)
kubectl get pods -A -o wide | grep -E "fcff::|fd20::"
```

Create Pods on two nodes and test connectivity:

```bash
kubectl run test-w1 --image=busybox \
    --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"worker-1.ryskn.k8s"}}}' \
    -- sleep 3600
kubectl run test-w2 --image=busybox \
    --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"worker-2.ryskn.k8s"}}}' \
    -- sleep 3600

W2_IP=$(kubectl get pod test-w2 -o jsonpath='{.status.podIP}')
kubectl exec test-w1 -- ping -c 5 "$W2_IP"
```

VPP SR state:

```bash
VPP=$(kubectl -n calico-vpp-dataplane get pod -l k8s-app=calico-vpp-node -o wide \
      | awk '/master/ {print $1; exit}')

kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr steering-policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr localsids
kubectl -n calico-vpp-dataplane exec -i $VPP -c agent -- gobgp neighbor
```

When healthy:
- SR policies: a BSID (`cafe::…`) and Segment Lists installed per node
- SR steering: Pod CIDR → BSID steering
- LocalSID: `End.DT6` SID, with the Good traffic counter increasing

## Summary

- It works with the **official v3.32.0 images** (PR #973 + #982 + VPP Gerrit 45604 are all merged; fork build is unnecessary)
- With IPv6 single-stack, `kubeadm init` is **OK with v6 advertise / v6 CIDRs** (= the dual-stack workaround required in older versions is no longer needed)
- Pod IPs are allocated not from `fd20::/64`, but from the `fcff:0:0:<node>AA::/64` LocalSID pools (= Pod IP = End.DT6 SID, the SRv6-native design)
- **Three AlmaLinux cloud-init traps**:
  - Use `manage_etc_hosts: false` so `/etc/hosts` is not reset every time
  - Use `chattr +i /etc/sysconfig/kubelet` to protect `--node-ip=fd00:1::*`
  - Use `chattr +i /etc/resolv.conf` to protect the internal DNS
- **Always disable `multicast_snooping=1` on Proxmox vmbr0**. Otherwise IPv6 ND cannot resolve, and packets never leave on the physical wire
- **Explicitly set the Service CIDR to `fd30::/108`**. If it remains at the default `10.96.0.0/12`, cnat DNAT cannot resolve, and kubelet cannot reach the apiserver
- Make sure internal DNS (NSD + unbound) and dnf-proxy auto-start after boot. If DNS fails to start after a reboot, every install fails in a chain reaction

---

PRs / Change (= how they were incorporated into v3.32.0):

- [projectcalico/vpp-dataplane#973](https://github.com/projectcalico/vpp-dataplane/pull/973) — three SR Policy receive-side fixes (two-stage BSID Unmarshal / SAFI 85 / injectRoute fallback)
- [projectcalico/vpp-dataplane#982](https://github.com/projectcalico/vpp-dataplane/pull/982) — cnat SNAT ENOENT idempotent guard (= temporary Go-side guard; the root fix is on the VPP side)
- [VPP Gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604) — makes the `cnat_snat_policy` delete path re-entrant. It preserves lazy init immediately after `pool_get_zero`, while also preventing ENOENT from being returned for delete-only sequences
