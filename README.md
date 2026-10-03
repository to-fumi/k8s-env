# k8s-env

macOS 上で Multipass の VM を使い、kubeadm で Kubernetes クラスタを構築する検証環境です。

## 構成

| VM | 役割 | CPU | メモリ | ディスク |
| --- | --- | --- | --- | --- |
| cp | コントロールプレーン | 2 | 4G | 20G |
| worker1 | ワーカー | 2 | 3G | 20G |
| worker2 | ワーカー | 2 | 3G | 20G |

OS は Ubuntu 24.04 です。

## 手順

1. [Multipass による VM 構築](docs/01-multipass-vm-setup.md)
2. [コンテナランタイムの導入](docs/02-container-runtime.md)
3. [kubeadm によるクラスタ初期化](docs/03-kubeadm-init.md)
4. [ワーカーノードの参加](docs/04-worker-join.md)

## ディレクトリ

- `docs/`: 構築手順書（番号順に実行）
- `cloud-init/`: VM 作成時に渡す cloud-init 設定
