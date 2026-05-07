---
title: "Calico-VPP + SRv6 を IPv6 single-stack で運用する"
emoji: "🛰️"
type: "tech"
topics: ["kubernetes", "calico", "vpp", "srv6", "ipv6"]
published: true
---

Proxmox 上の 3 ノード VM (master×1, worker×2) に Calico-VPP の SRv6 データプレーンを IPv6 single-stack 構成でセットアップする手順。

SRv6 は underlay が IPv6 前提なので、Pod / Service も v6 に揃えた方が自然に動く。前に dual-stack で組んだら cnat 経由の IPv4 Service が通らなかったり、agent が cnat の ENOENT で落ちたりして、v4 は実装の「おまけ」感が拭えなかったので単一 family に寄せる。

:::message
**v3.32.0 で動くのは:** ここで書いた手順は Calico-VPP **v3.32.0** (= projectcalico/vpp-dataplane 公式 release) で動かしたもの。**fork build 不要** で公式 image (`docker.io/calicovpp/{vpp,agent}:v3.32.0`) で動く。それ以前の 3.31.x までは SRv6 install 周りの 3 件の不具合 ([PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)) と、v6 single-stack で agent が落ちる cnat ENOENT 問題 ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982) + [VPP gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604)) があり、自前 image build が必要だった。v3.32.0 でこれらは全部 merged 済。
:::

## 構成

| Component | IPv4 (mgmt) | IPv6 ULA (k8s control-plane) |
|---|---|---|
| **infra01** (DNS / dnf-proxy) | 192.168.1.6 | (auto SLAAC) |
| **master** | 192.168.1.10 | fd00:1::10 |
| **worker-1** | 192.168.1.11 | fd00:1::11 |
| **worker-2** | 192.168.1.12 | fd00:1::12 |

- 管理 NW: `192.168.1.0/24` (IPv4)
- IPv6 underlay: Proxmox host の RA 経由で `240b:…/64` SLAAC + 静的 ULA (`fd00:1::/64`) を eth0 に追加
- 内部 DNS: infra01 上で **NSD (authoritative for `ryskn.k8s`)** + **unbound (recursive resolver)**
- ローカル dnf proxy: infra01 上の nginx (`http://dnf-proxy.ryskn.k8s/almalinux/...`)
- Pod CIDR: `fd20::/64` (Installation CR で IPPool として明示)
- Service CIDR: `fd30::/108`
- Pod 割当: **`fcff:0:0:<per-node>AA::/64` の LocalSID プール** (Pod IP = End.DT6 SID)

## なぜ IPv6 single-stack か

cnat の IPv4 経路は IPv6 single-stack だと「INCLUDE_V4 が空」になり、agent reconcile loop で disable を投げると ENOENT で agent が落ちる ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982))。dual-stack でも cnat の IPv4 NAT は不安定で、Service の半数が通らないことがある。SRv6 は underlay が v6 前提なので、Pod / Service / control-plane を v6 に揃える方が trip-wire が減る。

## 0. infra (DNS / dnf-proxy) の前提

`ryskn.k8s` zone を返す internal DNS server が居ないと、`master-1.ryskn.k8s` や `dnf-proxy.ryskn.k8s` が引けなくて先に進めない。

infra01 (= 192.168.1.6) に:
- **NSD** で `ryskn.k8s` zone を authoritative に serve (例: `dnf-proxy IN A 192.168.1.6`)
- **unbound** で recursive resolver、`ryskn.k8s` だけ `127.0.0.1@1053` (= NSD) に stub-zone forward、それ以外は `1.1.1.1` に forward
- nginx で `/almalinux/`、`/cri-o/` 等を AlmaLinux 公式 mirror から proxy (= ローカル dnf-proxy)

両方が `systemctl enable --now` されてること。**boot 後に自動起動を忘れがち** で、reboot 後に DNS が無くて全部死ぬ。

## 1. Proxmox VM 作成

Proxmox host で AlmaLinux 10 cloud-init テンプレ (VMID 9001) からクローン:

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

# worker-2 (VMID 102) も同様 (ip=192.168.1.12)

qm start 103 && qm start 101 && qm start 102
```

## 2. Proxmox vmbr0 の multicast snooping を無効化

**最大級のハマりどころ**。Proxmox のデフォルト設定では vmbr0 に `multicast_snooping=1` が有効かつ multicast querier が居ないため、VM 間で IPv6 の Neighbor Solicitation (`ff02::1:ff…`) が flood されない。VPP は SRv6 encap 後のパケットを uplink global v6 宛に投げようとするが ND が解決できず、物理線に一切出ないまま詰まる。

```bash
# 確認 (デフォルトは 1)
cat /sys/class/net/vmbr0/bridge/multicast_snooping

# 即効適用
echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping

# 永続化: /etc/network/interfaces の vmbr0 ブロックに追記
#   post-up echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping
```

## 3. AlmaLinux cloud-init の罠 (3 つ)

**今回最も悩まされた**。AlmaLinux cloud-init テンプレを使うと、reboot のたびに以下が re-set される:

### 3-1. `manage_etc_hosts: True` で `/etc/hosts` が再生成される

`fd00:1::10 master-1.ryskn.k8s` 等の手動エントリが boot 毎に消える。

**対処**: `/etc/cloud/cloud.cfg` で `manage_etc_hosts: false` に。

### 3-2. `/etc/sysconfig/kubelet` が空に reset される

`KUBELET_EXTRA_ARGS=--node-ip=fd00:1::10 ...` が空に書き換えられて、kubelet が IPv4 の InternalIP で再 register される (= cluster の v6 single-stack 設計と矛盾)。

**対処**: `chattr +i /etc/sysconfig/kubelet` で immutable に。

### 3-3. NetworkManager が `/etc/resolv.conf` を再生成

`nameserver 192.168.1.6` (内部 DNS) を書いても消えて、結果 `dnf-proxy.ryskn.k8s` が引けなくて全 install が死ぬ。

**対処**: `chattr +i /etc/resolv.conf` で immutable に。

## 4. 全ノード共通の下準備

各ノード (master + worker×2) に root で SSH (ユーザー名 `ryosuke`、`sudo bash` で実行):

```bash
# === [0/6] kubernetes / cri-o repos (公式 pkgs.k8s.io) ===
# kubernetes 1.36 / cri-o 1.32 stable. cri-o 1.33+ は stable 未公開で 1.32 に揃える
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

# === [0.5/6] DNS (chattr +i で固定) ===
chattr -i /etc/resolv.conf 2>/dev/null || true
printf "nameserver 192.168.1.6\nsearch ryskn.k8s\n" > /etc/resolv.conf
chattr +i /etc/resolv.conf

# === [0.6/6] eth0 に ULA IPv6 を追加 ===
# IPv6 single-stack k8s で apiserver は v6 IP に bind する必要があり、
# Proxmox の SLAAC v6 (240b::/64) は VPP に取られるので、各ノードに静的 ULA を追加。
HOST_LAST_OCTET=$(ip -4 -o addr show eth0 | awk '{print $4}' | cut -d. -f4 | cut -d/ -f1)
ULA_ADDR="fd00:1::${HOST_LAST_OCTET}/64"
NM_CONN=$(nmcli -g NAME,DEVICE con show --active | awk -F: '$2=="eth0"{print $1; exit}')
nmcli con mod "$NM_CONN" +ipv6.addresses "$ULA_ADDR"
nmcli con mod "$NM_CONN" ipv6.method auto
nmcli con up "$NM_CONN"

# === [0.7/6] /etc/hosts (cloud-init 干渉を disable) ===
sed -i 's/^manage_etc_hosts:.*/manage_etc_hosts: false/' /etc/cloud/cloud.cfg
grep -q '^manage_etc_hosts:' /etc/cloud/cloud.cfg || \
    echo 'manage_etc_hosts: false' >> /etc/cloud/cloud.cfg
sed -i '/ryskn.k8s$/d' /etc/hosts
cat >> /etc/hosts <<EOF
fd00:1::10 master-1.ryskn.k8s
fd00:1::11 worker-1.ryskn.k8s
fd00:1::12 worker-2.ryskn.k8s
EOF

# === [0.8/6] v6 Service CIDR の dummy route ===
# kube-proxy DNAT が OUTPUT chain で動くために必要
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

# === [6/6] kubeadm / kubelet / kubectl + cloud-init 干渉防止 ===
dnf install -y kubeadm kubelet kubectl

NODE_ULA="fd00:1::${HOST_LAST_OCTET}"
chattr -i /etc/sysconfig/kubelet 2>/dev/null || true
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///var/run/crio/crio.sock --node-ip=${NODE_ULA}
EOF
chattr +i /etc/sysconfig/kubelet  # cloud-init で空に書き換えられるのを防ぐ

systemctl daemon-reload && systemctl enable kubelet
```

:::message
**`KUBELET_EXTRA_ARGS` を `/etc/systemd/system/kubelet.service.d/20-crio.conf` の `Environment=` ではなく `/etc/sysconfig/kubelet` に書く**。kubeadm の標準 drop-in (`/usr/lib/.../10-kubeadm.conf`) が `EnvironmentFile=-/etc/sysconfig/kubelet` を読んでて、こちらの値が drop-in の `Environment=` を上書きするため。drop-in に書いても効かない。
:::

## 5. master 初期化

master (192.168.1.10) で:

```bash
sudo kubeadm init \
    --cri-socket=unix:///var/run/crio/crio.sock \
    --apiserver-advertise-address=fd00:1::10 \
    --pod-network-cidr=fd20::/64 \
    --service-cidr=fd30::/108 \
    --control-plane-endpoint=master-1.ryskn.k8s

# kubeconfig (root + ryosuke 両方)
mkdir -p /root/.kube && cp -f /etc/kubernetes/admin.conf /root/.kube/config
mkdir -p /home/ryosuke/.kube && cp -f /etc/kubernetes/admin.conf /home/ryosuke/.kube/config
chown -R ryosuke:ryosuke /home/ryosuke/.kube

sudo kubeadm token create --print-join-command > /home/ryosuke/k8s-join.sh
chown ryosuke:ryosuke /home/ryosuke/k8s-join.sh
```

## 6. worker join

各 worker で:

```bash
JOIN_CMD=$(ssh ryosuke@192.168.1.10 'cat ~/k8s-join.sh')
sudo bash -c "$JOIN_CMD --cri-socket=unix:///var/run/crio/crio.sock"
```

確認:

```bash
kubectl get nodes -o wide
# NAME                 STATUS     INTERNAL-IP   VERSION
# master.ryskn.k8s     Ready      fd00:1::10    v1.36.0
# worker-1.ryskn.k8s   NotReady   fd00:1::11    v1.36.0
# worker-2.ryskn.k8s   NotReady   fd00:1::12    v1.36.0
```

`NotReady` は CNI 未 install で正常 (= 次の section で直る)。InternalIP が **v6** になってれば `chattr +i /etc/sysconfig/kubelet` が効いてる証拠。

## 7. tigera-operator + Calico Installation (v3.32.0)

master で:

```bash
# operator-crds + tigera-operator (v3.32.0)
kubectl apply --server-side -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/operator-crds.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
kubectl -n tigera-operator rollout status deployment/tigera-operator --timeout=300s

until kubectl get crd installations.operator.tigera.io >/dev/null 2>&1; do sleep 5; done
```

Installation CR を **v6 ipPools のみ明示** で apply (= operator が v4 pool を勝手に作るのを抑止):

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

## 8. Calico-VPP マニフェスト (kustomize)

リポジトリの `yaml/overlays/` に自前 overlay を作る。base は `test-vagrant` (= 公式の汎用 base)。

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

### 8-3. srv6-cm-patch.yaml (SRv6 機能 ON + Service CIDR を v6 に)

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
**`SERVICE_PREFIX` をデフォルト (`10.96.0.0/12`) のままにしないこと**。v6 single-stack で apiserver Service IP は `fd30::1` なので、cnat が DNAT を解決できなくて kubelet が apiserver に到達できない。今回の cluster ではこれで丸 1 日溶かした。
:::

### 8-4. srv6res-patch.yaml (LocalSID プール)

LocalSID プールはノード名で厳密にマッチさせる (`sr-localsids-pool-<hostname>`)。Calico-VPP agent はこの命名規約でプールを検索する。

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

### 8-5. rbac-patch.yaml (ipreservations 権限)

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

### 8-6. image-patch.yaml (公式 v3.32.0 image を指定)

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
**v3.32.0 公式 image** で動く。以前は SRv6 install 周りの 3 件のバグ ([PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)) と cnat ENOENT ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982) + [VPP gerrit #45604](https://gerrit.fd.io/r/c/vpp/+/45604)) があり fork build が必要だったが、v3.32.0 で全て merged 済。
:::

### 8-7. 生成と適用

```bash
kubectl kustomize yaml/overlays/dev-ryskn-srv6 > calico-vpp-srv6.yaml
kubectl apply -f calico-vpp-srv6.yaml
```

## 9. 動作確認

```bash
# ノード Ready
kubectl get nodes -o wide
# NAME                 STATUS  INTERNAL-IP
# master.ryskn.k8s     Ready   fd00:1::10
# worker-1.ryskn.k8s   Ready   fd00:1::11
# worker-2.ryskn.k8s   Ready   fd00:1::12

# Calico-VPP DaemonSet が 2/2 Running、image が v3.32.0
kubectl -n calico-vpp-dataplane get ds calico-vpp-node -o wide

# Pod IP が SRv6 LocalSID プール (fcff:0:0:<node>AA::/64) から取れてる
kubectl get pods -A -o wide | grep -E "fcff::|fd20::"
```

Pod を 2 ノードに作って疎通:

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

VPP の SR 状態:

```bash
VPP=$(kubectl -n calico-vpp-dataplane get pod -l k8s-app=calico-vpp-node -o wide \
      | awk '/master/ {print $1; exit}')

kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr steering-policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr localsids
kubectl -n calico-vpp-dataplane exec -i $VPP -c agent -- gobgp neighbor
```

正常時:
- SR policies: ノード数 × 2 (v4/v6) の BSID (`cafe::…`) + Segment Lists がインストール済
- SR steering: Pod CIDR → BSID の steering
- LocalSID: `DT4` と `DT6` の End SID、Good traffic counter が増えていく

## まとめ

- **v3.32.0 公式 image** で動く (PR #973 + #982 + VPP gerrit 45604 全部 merged 済、fork build 不要)
- IPv6 single-stack で kubeadm init は **v6 advertise / v6 CIDR で OK** (= 旧版で必要だった dual-stack 経由は不要に)
- Pod IP は `fd20::/64` ではなく `fcff:0:0:<node>AA::/64` の LocalSID プールから割当 (= Pod IP = End.DT6 SID、SRv6 native 設計)
- **AlmaLinux cloud-init の罠 3 点**:
  - `manage_etc_hosts: false` で `/etc/hosts` を毎回 reset させない
  - `chattr +i /etc/sysconfig/kubelet` で `--node-ip=fd00:1::*` を保護
  - `chattr +i /etc/resolv.conf` で内部 DNS を保護
- **Proxmox vmbr0 の `multicast_snooping=1` は必ず落とす**。落とさないと IPv6 ND が解けず物理線に出ない
- **Service CIDR を `fd30::/108` に明示**。default の `10.96.0.0/12` のままだと cnat DNAT が解決できず kubelet が apiserver 不通
- 内部 DNS (NSD + unbound) と dnf-proxy が boot 後 auto-start されてること。reboot 後に DNS が起動失敗してると全 install が連鎖死亡

---

PR / Change (= v3.32.0 に取り込み済の経緯):

- [projectcalico/vpp-dataplane#973](https://github.com/projectcalico/vpp-dataplane/pull/973) — SR Policy 受信周りの 3 件 (BSID 2 段 Unmarshal / SAFI 85 / injectRoute fallback)
- [projectcalico/vpp-dataplane#982](https://github.com/projectcalico/vpp-dataplane/pull/982) — cnat SNAT ENOENT idempotent guard (= 暫定 Go 側 guard、根本修正は VPP 側)
- [VPP gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604) — `cnat_snat_policy` の del path を re-entrant に。`pool_get_zero` 直後の lazy init を維持しつつ、del-only 系列でも ENOENT を返さないように
