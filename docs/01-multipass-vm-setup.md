# Multipass による VM 構築

コマンドはリポジトリのルート（k8s-env/）で実行します。

## 1. Multipass のインストール

```bash
brew install --cask multipass
```

## 2. cloud-init ファイルの作成

kubeadm が必須とするカーネル設定を、VM 作成時に3台へ同じように自動適用するためです。br_netfilter・bridge-nf-call は Pod 間のブリッジ通信を iptables（kube-proxy）に通すため、ip_forward はノード間で Pod 通信を転送するため、swap 無効化は kubelet のメモリ管理の前提のために必要です。

```bash
mkdir -p cloud-init
cat <<'EOF' > cloud-init/k8s-prep.yaml
#cloud-config
write_files:
  - path: /etc/modules-load.d/k8s.conf
    content: |
      overlay
      br_netfilter
  - path: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
runcmd:
  - modprobe overlay
  - modprobe br_netfilter
  - sysctl --system
  - swapoff -a
EOF
```

## 3. VM の作成

kubeadm は CPU 2 コア・メモリ 2GB 未満だと preflight で失敗するため、全台 2 コア以上にしています。cp は etcd や API サーバーが載るのでメモリを多めにしています。

1台目は単独で実行してイメージをキャッシュさせ、残り2台は & でバックグラウンド実行、wait で両方の完了を待ちます。

```bash
multipass launch 24.04 --name cp --cpus 2 --memory 4G --disk 20G --cloud-init cloud-init/k8s-prep.yaml

multipass launch 24.04 --name worker1 --cpus 2 --memory 3G --disk 20G --cloud-init cloud-init/k8s-prep.yaml &
multipass launch 24.04 --name worker2 --cpus 2 --memory 3G --disk 20G --cloud-init cloud-init/k8s-prep.yaml &
wait
```

3台の状態と IP を確認します。

```bash
multipass list
```

## 4. 起動トラブル時

Stopped になった VM は起動し直します。

```bash
multipass start cp
```

## 5. VM 内のカーネル設定の確認・修正

cloud-init はエラーがあっても VM の起動自体は成功するため、設定が実際に反映されたかを確認します。ファイルへの書き込みは再起動後も設定を残すため、modprobe・sysctl は今すぐ反映させるために行います。

```bash
multipass shell cp
```

```bash
cat /etc/sysctl.d/k8s.conf
cat /etc/modules-load.d/k8s.conf
```

```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system > /dev/null
```

```bash
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
lsmod | grep br_netfilter
```

終わったら抜けて、worker1・worker2 でも同じ手順を行います。

```bash
exit
```

## 6. /etc/hosts の設定

Mac 側で3台の IP を確認します。

```bash
multipass list
```

3台すべての VM 内の /etc/hosts に、3台分の「IP 名前」を追記します。cp の行は kubeadm・kubelet・kubectl が API サーバーに接続するのに使い、worker の行は作業用です。-a を忘れると /etc/hosts が上書きされるので注意してください。

```bash
cat <<'EOF' | sudo tee -a /etc/hosts
192.168.64.x  cp
192.168.64.y  worker1
192.168.64.z  worker2
EOF
```

```bash
ping -c 2 cp
```

## 7. IP についての注意

Multipass の VM の IP は DHCP で割り当てられ、再起動などで変わることがあります。IP を直接指定すると証明書や設定ファイルに IP が埋め込まれて作り直しが必要になるため、名前で指定しておき IP が変わっても /etc/hosts の修正だけで済むようにします。

CRI-O と kubeadm・kubelet・kubectl をインストールし、cp で名前を使って初期化します。

```bash
sudo kubeadm init --control-plane-endpoint=cp:6443
```
