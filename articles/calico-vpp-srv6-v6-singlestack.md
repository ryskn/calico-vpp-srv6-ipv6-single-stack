---
title: "Calico-VPP + SRv6 を IPv6 single-stack で運用する"
emoji: "🛰️"
type: "tech"
topics: ["kubernetes", "calico", "vpp", "srv6", "ipv6"]
published: true
---

Proxmox 上の 3 ノード VM (master×1, worker×2) に Calico-VPP の SRv6 データプレーンを IPv6 single-stack 構成でセットアップする手順。BGP peer は v4/v6 両方で冗長化する。

SRv6 は underlay が IPv6 前提なので、Pod / Service も v6 に揃えた方が自然に動く。前に dual-stack で組んだら cnat 経由の IPv4 Service が ~50% 通らず、v4 は実装の「おまけ」感が拭えなかったので、今回は単一 family に寄せる。

## 構成

- **master-1**: 2 vCPU / 4 GB / NIC×1 (vmbr0), `192.168.1.102`
- **worker-1, worker-2**: 4 vCPU / 8 GB / NIC×1 (vmbr0), `192.168.1.103`, `192.168.1.104`
- 管理 NW: `192.168.1.0/24` (IPv4)
- IPv6 underlay: Proxmox host の RA 経由で `240b:…/64` SLAAC
- DNS: `192.168.1.101` (内部 DNS)
- ローカルリポジトリ: `http://local.repo.ryskn.k8s/` (RPM + image tarball)
- Pod CIDR (kubeadm): `172.16.0.0/16,fd20::/64` (dual-stack で init、実割当は IPPool 側で v6 に制限)
- Service CIDR (kubeadm): `10.96.0.0/12,fd30::/108`
- Pod 割当: **`fcff:0:0:<per-node>AA::/64` の LocalSID プール** (Pod IP = End.DT6 SID)

:::message
kubeadm init だけは dual-stack CIDR のままにする。v6 single-stack CIDR だと apiserver が「service CIDR の family と自ノード IP の family が一致しない」と言って起動しないため。実際の Pod 割当は IPPool で絞る。
:::

## なぜ BGP peer を v4 と v6 両方張るのか

- v4 peer: kernel netns 側の link-local 問題で失敗しがちだった v6 peer が落ちても、経路配布が止まらない冗長経路として機能する。node IP が v4 なので auto-mesh で自動的に張られる
- v6 peer: SRv6 dataplane 自体は v6 underlay なので、control plane も v6 で peer しておくと transport を揃えられる
- 両方 Established になっていれば、mp-bgp で v4/v6 両 AFI の routes を冗長に交換できる

## 1. Proxmox VM 作成

Proxmox host で AlmaLinux 10 cloud-init テンプレ (VMID 9001) からクローン:

```bash
# master-1
qm clone 9001 102 --name master-1 --full
qm set 102 --cores 2 --sockets 1 --cpu host --memory 4096 --balloon 0 \
    --net0 virtio,bridge=vmbr0 --agent enabled=1 --onboot 1
qm resize 102 scsi0 40G
qm set 102 --ipconfig0 ip=192.168.1.102/24,gw=192.168.1.1,ip6=auto
qm set 102 --nameserver 192.168.1.101 --searchdomain ryskn.k8s

# worker-1
qm clone 9001 103 --name worker-1 --full
qm set 103 --cores 4 --sockets 1 --cpu host --memory 8192 --balloon 0 \
    --net0 virtio,bridge=vmbr0 --agent enabled=1 --onboot 1
qm resize 103 scsi0 60G
qm set 103 --ipconfig0 ip=192.168.1.103/24,gw=192.168.1.1,ip6=auto
qm set 103 --nameserver 192.168.1.101 --searchdomain ryskn.k8s

# worker-2 も同様 (VMID 104, IP 192.168.1.104)

qm start 102 && qm start 103 && qm start 104
```

## 2. Proxmox vmbr0 の multicast snooping を無効化

**これが最大のハマりどころ**。Proxmox のデフォルト設定では vmbr0 に `multicast_snooping=1` が有効かつ multicast querier が居ないため、VM 間で IPv6 の Neighbor Solicitation (`ff02::1:ff…`) が flood されない。VPP は SRv6 encap 後のパケットを uplink global v6 宛に投げようとするが ND が解決できず、物理線に一切出ないまま詰まる。

Proxmox host で:

```bash
# 確認 (デフォルトは 1 / 0)
cat /sys/class/net/vmbr0/bridge/multicast_snooping
cat /sys/class/net/vmbr0/bridge/multicast_querier

# 即効適用
echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping

# 永続化: /etc/network/interfaces の vmbr0 ブロックに追記
#   post-up echo 0 > /sys/class/net/vmbr0/bridge/multicast_snooping
```

適用後、ノード間の ND が解けるようになる。構築完了してから「Pod-to-Pod が片方向だけ動く」みたいな怪奇現象に出くわしたらまずここを疑う。

## 3. 全ノード共通の下準備

各ノード (master + worker×2) に root で SSH して実行:

```bash
# ローカルリポジトリ追加
cat > /etc/yum.repos.d/local-extra.repo <<EOF
[local-cri-o]
name=Local cri-o 1.35
baseurl=http://local.repo.ryskn.k8s/cri-o/1.35/
enabled=1
gpgcheck=0

[local-kubernetes]
name=Local Kubernetes 1.35
baseurl=http://local.repo.ryskn.k8s/kubernetes/1.35/
enabled=1
gpgcheck=0
EOF
dnf clean all && dnf makecache

# DNS
printf "nameserver 192.168.1.101\nsearch ryskn.k8s\n" > /etc/resolv.conf

# swap off
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# SELinux permissive
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# カーネルモジュール (br_netfilter は kernel-modules-extra に含まれる)
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

# sysctl
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.forwarding        = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.conf.default.rp_filter     = 0
net.ipv4.conf.eth0.rp_filter        = 0
EOF

# hugepages (VPP が 512 MB 要求するので倍取る)
cat > /etc/sysctl.d/hugepages.conf <<EOF
vm.nr_hugepages = 512
EOF
sysctl --system

# cri-o
dnf install -y cri-o
systemctl enable --now crio

# kubeadm/kubelet/kubectl
dnf install -y kubeadm kubelet kubectl podman
mkdir -p /etc/systemd/system/kubelet.service.d
cat > /etc/systemd/system/kubelet.service.d/20-crio.conf <<EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///var/run/crio/crio.sock"
EOF
systemctl daemon-reload
systemctl enable kubelet
```

## 4. master 初期化

master (192.168.1.102) で:

```bash
kubeadm init \
    --cri-socket=unix:///var/run/crio/crio.sock \
    --pod-network-cidr=172.16.0.0/16,fd20::/64 \
    --service-cidr=10.96.0.0/12,fd30::/108 \
    --control-plane-endpoint=master-1.ryskn.k8s

mkdir -p ~/.kube
cp -f /etc/kubernetes/admin.conf ~/.kube/config

kubeadm token create --print-join-command > /root/k8s-join.sh
```

dual-stack CIDR で init する (apiserver の bind 互換のため)。Pod 割当を v6 に絞るのは後の Installation CR で行う。

## 5. worker join

各 worker で master が出力した join コマンドを実行:

```bash
kubeadm join master-1.ryskn.k8s:6443 \
    --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH> \
    --cri-socket=unix:///var/run/crio/crio.sock
```

## 6. tigera-operator と Installation CR (v6 IPPool のみ明示)

master で:

```bash
# CRD (annotation サイズ大のため server-side)
curl -sLO https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/operator-crds.yaml
kubectl apply --server-side -f operator-crds.yaml

# operator 本体
curl -sLO https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
kubectl apply --server-side -f tigera-operator.yaml
kubectl -n tigera-operator rollout status deployment/tigera-operator --timeout=300s

until kubectl get crd installations.operator.tigera.io >/dev/null 2>&1; do sleep 5; done
```

**ここが 2 つめのハマりどころ**。Installation CR で `ipPools` を明示しないと、tigera-operator は kubeadm の `pod-network-cidr` を見て `default-ipv4-ippool` と `default-ipv6-ippool` を勝手に作る。dual-stack CIDR で init しているので v4 pool まで生えて、Pod が v4 で割り当てられてしまう。

v6 pool のみを **明示的に** 列挙する:

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

ipPools を省略した spec で apply してしまった後でこれを上書きしても、operator は v4 pool を即座には消さない。既に作られてたら明示削除する:

```bash
kubectl delete ippool default-ipv4-ippool
```

## 7. Calico-VPP マニフェスト (kustomize)

リポジトリの `yaml/overlays/dev-ryskn-srv6/` (あるいは自分の overlay ディレクトリ) に以下を配置する。

### 7-1. kustomization.yaml

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

### 7-2. config-patch.yaml (VPP uplink 設定)

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

### 7-3. srv6res-patch.yaml (IPPool v6 のみ)

LocalSID プールはノード名で厳密にマッチさせる (`sr-localsids-pool-<hostname>`)。Calico-VPP agent がこの命名規約でプールを検索している。

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
    name: sr-localsids-pool-master-1.ryskn.k8s
  spec:
    cidr: fcff:0:0:00AA::/64
    ipipMode: Never
    nodeSelector: kubernetes.io/hostname == 'master-1.ryskn.k8s'
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

### 7-4. rbac-patch.yaml (ipreservations 権限を追加)

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

### 7-5. image-patch.yaml (fix 入り image を指定)

PR #973 と PR #982 の fix を両方含んだ image を GHCR に push 済みなので、それを直接 pull する。tag は commit SHA を使う:

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
        image: ghcr.io/ryskn/calicovpp/vpp:beb8ef38bcacae9caa4369d514c637587ca5ac28
        imagePullPolicy: IfNotPresent
      - name: agent
        image: ghcr.io/ryskn/calicovpp/agent:beb8ef38bcacae9caa4369d514c637587ca5ac28
        imagePullPolicy: IfNotPresent
```

この image に含まれている fix:

- **[PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)** (merged): SR Policy install を直す 3 件 — BSID の `anypb.Any` 2 段 Unmarshal、SR Policy SAFI (`85`) を受け付けるよう条件修正、`injectRoute` fallback の抑止
- **[PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982)** (open): IPv6 single-stack で発生する cnat SNAT の ENOENT idempotency 問題のガード。`cnat_snat_policy_add_del_if(is_add=false)` が一度も add されていない family (= INCLUDE_V4 が空) で発火すると VPP API が `-6 No such entry` を返し、agent の reconcile loop が `dying` に入るのを防ぐ
  - 根本修正は VPP 側で対応済み ([gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604))。これがリリースに入るまでの暫定 guard

multi-arch (amd64 / arm64) で push してあるので、x86 / ARM 両方の cluster で同じ tag を使える。

### 7-6. 生成と適用

```bash
kubectl kustomize yaml/overlays/dev-ryskn-srv6 > calico-vpp-srv6.yaml
kubectl apply -f calico-vpp-srv6.yaml
```

## 8. fix 入り VPP / Agent image を入手する

7-5 で指定した image は **公式 release (v3.31.x) ではなく** PR #973 + PR #982 を当てたもの。release 版には SRv6 dataplane の不具合 3 件 (BSID の 2 段 Unmarshal / SAFI `85` 未対応 / `injectRoute` fallback の干渉) と、IPv6 single-stack で agent を巻き込む cnat ENOENT があるため、そのままだと SR Policy が install されないか、再起動時に agent が落ちる。

入手方法は 2 つある。

### 8-a. GHCR から pull する (推奨)

7-5 の `image-patch.yaml` に書いた tag をそのまま使う。各ノードに事前 pull したい場合:

```bash
for h in 192.168.1.102 192.168.1.103 192.168.1.104; do
    ssh root@$h '
        podman pull ghcr.io/ryskn/calicovpp/vpp:beb8ef38bcacae9caa4369d514c637587ca5ac28
        podman pull ghcr.io/ryskn/calicovpp/agent:beb8ef38bcacae9caa4369d514c637587ca5ac28
    '
done
```

`imagePullPolicy: IfNotPresent` にしているので、最初の Pod 起動時に kubelet が pull する。pre-pull はオフライン耐性が欲しいときだけで OK。

GHCR public で配布しているので auth は不要。private にした場合は `imagePullSecret` を `calico-vpp-dataplane` namespace に置く必要がある。

### 8-b. 自前で master ブランチからビルドする (代替)

自分で fix を当て直したい / patch を変えたい場合はこちら。

```bash
git clone https://github.com/projectcalico/vpp-dataplane.git
cd vpp-dataplane
# PR #982 を取り込む (まだ未 merge なので必要)
git fetch origin pull/982/head:pr982
git merge --no-edit pr982

make -C vpp-manager vpp-image TAG=srv6-fix
make -C calico-vpp-agent image TAG=srv6-fix
```

成果物 (`calicovpp/vpp:srv6-fix` / `calicovpp/agent:srv6-fix`) を tarball で配布:

```bash
podman save -o /tmp/calicovpp-vpp-srv6-fix.tar calicovpp/vpp:srv6-fix
podman save -o /tmp/calicovpp-agent-srv6-fix.tar calicovpp/agent:srv6-fix

for h in 192.168.1.102 192.168.1.103 192.168.1.104; do
    scp /tmp/calicovpp-*-srv6-fix.tar root@$h:/tmp/
    ssh root@$h '
        podman load -i /tmp/calicovpp-vpp-srv6-fix.tar
        podman load -i /tmp/calicovpp-agent-srv6-fix.tar
    '
done
```

この場合は 7-5 の `image-patch.yaml` を `image: calicovpp/vpp:srv6-fix` / `imagePullPolicy: Never` に書き戻す。

## 9. Calico 本体 image の配布

VPP 以外の Calico コンポーネント (node / typha / kube-controllers / apiserver / csi / tigera-operator) は公式 release で問題ないので、全ノードに普通にロードする:

```bash
for t in tigera-operator-v1.40.7 calico-node-v3.31.4 calico-cni-v3.31.4 \
         calico-typha-v3.31.4 calico-kube-controllers-v3.31.4 \
         calico-apiserver-v3.31.4 calico-csi-v3.31.4 \
         calico-node-driver-registrar-v3.31.4; do
    curl -fsSLO http://local.repo.ryskn.k8s/k8s-images/${t}.tar
    podman load -i ${t}.tar
done
```

タグが落ちる場合は `podman tag <SHA> docker.io/calico/node:v3.31.4` のように付け直す。

## 10. 動作確認

```bash
# ノード Ready
kubectl get nodes -o wide
# INTERNAL-IP 列は v4 で OK (node IP は v4 のまま、Pod/Service が v6)

# Calico-VPP DaemonSet が 2/2 Running、image が :srv6-fix
kubectl -n calico-vpp-dataplane get ds calico-vpp-node -o wide

# IPPool: default-ipv6-ippool + sr-policies-pool + sr-localsids-pool-* のみ
kubectl get ippool
```

Pod を 2 ノードに作って疎通を確認する:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-w1
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-1.ryskn.k8s
  containers:
  - name: test
    image: docker.io/library/busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0
      capabilities:
        add: ["NET_RAW", "NET_ADMIN"]
---
apiVersion: v1
kind: Pod
metadata:
  name: test-w2
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-2.ryskn.k8s
  containers:
  - name: test
    image: docker.io/library/busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0
      capabilities:
        add: ["NET_RAW", "NET_ADMIN"]
EOF

kubectl get pods -o wide
```

Pod IP が `fcff::11aa:…` / `fcff::12aa:…` のように **各ノードの LocalSID プールから v6 で割り当てられている** ことを確認する。これは Calico-VPP SRv6 の設計で、Pod IP 自体が End.DT6 SID になっている。

ping と TCP を通す:

```bash
# worker-1 pod → worker-2 pod
kubectl exec test-w1 -- ping -c 5 <test-w2 の v6 IP>

# TCP 疎通
kubectl exec test-w2 -- sh -c "nohup nc -l -p 8080 -e /bin/date > /tmp/nc.log 2>&1 &"
sleep 1
kubectl exec test-w1 -- nc -w 3 <test-w2 の v6 IP> 8080
```

VPP の SR 状態:

```bash
VPP=$(kubectl -n calico-vpp-dataplane get pod -l k8s-app=calico-vpp-node -o wide \
      | grep master-1 | awk '{print $1}')

kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr steering-policies
kubectl -n calico-vpp-dataplane exec -i $VPP -c vpp -- vppctl show sr localsids

# BGP peer
kubectl -n calico-vpp-dataplane exec -i $VPP -c agent -- gobgp neighbor
```

正常時の出力:

- SR policies: ノード数 × 2 (v4/v6) の BSID (`cafe::…`) + Segment Lists がインストール済み
- SR steering: Pod CIDR (v4/v6 両方) → BSID の steering
- LocalSID: `DT4` と `DT6` の End SID が 1 つずつ、Good traffic counter が増えていく
- BGP: **v4 peer と v6 peer の両方が `Establ`**。ここが両方 Up していれば冗長化成立

## まとめ

- kubeadm は dual-stack CIDR で init する (apiserver の bind 互換のため)
- 実際の Pod 割当は Installation CR の `ipPools` を v6 のみに明示して絞る。省略すると operator が v4 pool を自動生成する
- Pod IP は `fd20::/64` ではなく `fcff:0:0:<node>AA::/64` の LocalSID プールから割当される。これが SRv6 の native な設計 (Pod IP = End.DT6 SID)
- BGP peer は node IP (v4) 経由の auto-mesh + VPP uplink の global v6 経由の両方が張られ、片方が落ちても経路配布が継続する
- Proxmox vmbr0 の `multicast_snooping=1` は必ず落とす。落とさないと IPv6 ND が解けず、SRv6 encap 後のパケットが物理線に出ない
- master ブランチの code ([PR #973](https://github.com/projectcalico/vpp-dataplane/pull/973)) + cnat ENOENT guard ([PR #982](https://github.com/projectcalico/vpp-dataplane/pull/982)) を当てた image を使う。release 版では SR Policy のインストールが壊れているのに加え、v6 single-stack だと agent が `cnat_snat_policy_add_del_if` の ENOENT で reconcile loop ごと落ちる

---

PR / Change:

- [projectcalico/vpp-dataplane#973](https://github.com/projectcalico/vpp-dataplane/pull/973) (merged) — SR Policy 受信周りの 3 件
- [projectcalico/vpp-dataplane#982](https://github.com/projectcalico/vpp-dataplane/pull/982) — cnat SNAT ENOENT idempotent guard (暫定)
- [VPP gerrit Change 45604](https://gerrit.fd.io/r/c/vpp/+/45604) — `cnat_snat_policy_entry` を `pool_get_zero` 直後に init して del-only 系列でも ENOENT を返さないようにする根本修正
