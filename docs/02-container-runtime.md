# CRI-O と kubeadm・kubelet・kubectl のインストール

cp・worker1・worker2 の3台すべての VM 内で実行します。

## 1. バージョンの指定

Kubernetes と CRI-O はマイナーバージョンを揃えてリリースされており、揃えておくと互換性の問題を避けられます。リポジトリ URL にバージョンが含まれるため、変数にしておき以降のコマンドで使い回します。

```bash
# 2026-10-03
KUBERNETES_VERSION=v1.37
CRIO_VERSION=v1.37
```

## 2. 必要なツールのインストール

curl はリポジトリの署名鍵のダウンロードに、gpg はその鍵を apt が読める形式に変換するために使います。

```bash
sudo apt-get update
sudo apt-get install -y curl gpg
```

## 3. 鍵の保存ディレクトリの作成

リポジトリの署名鍵は /etc/apt/keyrings に置くのが推奨です。古いバージョンの Ubuntu ではこのディレクトリが存在しないため、念のため作成しておきます。

```bash
sudo mkdir -p /etc/apt/keyrings
```

## 4. Kubernetes リポジトリの追加

kubelet・kubeadm・kubectl は Ubuntu 標準のリポジトリには含まれていないため、公式リポジトリ（pkgs.k8s.io）を追加します。署名鍵を signed-by で指定することで、このリポジトリのパッケージだけをこの鍵で検証し、改ざんされたパッケージのインストールを防ぎます。

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## 5. CRI-O リポジトリの追加

kubelet はコンテナを直接動かさず、CRI（Container Runtime Interface）を通してコンテナランタイムに依頼します。そのランタイムとして CRI-O を使うため、同様に CRI-O のリポジトリを追加します。

```bash
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    sudo tee /etc/apt/sources.list.d/cri-o.list
```

## 6. パッケージのインストール

追加したリポジトリを apt に認識させるため、update してからインストールします。cri-o はコンテナランタイム、kubelet は各ノードで Pod を起動・管理するエージェント、kubeadm はクラスタの初期化・参加を行うツール、kubectl はクラスタを操作する CLI です。

```bash
sudo apt-get update
sudo apt-get install -y cri-o kubelet kubeadm kubectl
```

## 7. バージョンの固定

apt upgrade などで意図せずバージョンが上がると、ノード間やコンポーネント間でバージョンがずれてクラスタが壊れることがあります。Kubernetes のアップグレードは kubeadm の手順に沿って行う必要があるため、hold で自動更新を止めておきます。

```bash
sudo apt-mark hold cri-o kubelet kubeadm kubectl
```

## 8. サービスの有効化

CRI-O は kubelet より先に動いている必要があるため、enable --now で今すぐ起動し、再起動後も自動起動するようにします。kubelet は kubeadm init / join で設定ファイルが作られるまで起動に失敗し続けるため、ここでは自動起動の設定（enable）だけにしておきます。

```bash
sudo systemctl enable --now crio
sudo systemctl enable kubelet
```

## 9. 参考サイト

[Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
[Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
