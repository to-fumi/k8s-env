# kubeadm と Cilium によるクラスタ構築

特に記載がない限り cp の VM 内で実行します。

## 1. コントロールプレーンの初期化

kubeadm init で API サーバー・etcd・scheduler・controller-manager などのコントロールプレーンを cp 上に立ち上げ、クラスタ内通信に使う証明書や kubeconfig を生成するためです。CRI-O だけがインストールされているため、CRI ソケットは自動で検出されます。

--skip-phases=addon/kube-proxy を付けると kube-proxy がデプロイされません。後の手順で Cilium に kube-proxy の役割（Service の通信の振り分け）を置き換えさせ、Cilium の Gateway API を使うためです。

```bash
# cp
sudo kubeadm init --skip-phases=addon/kube-proxy
```

--control-plane-endpoint を付けると、API サーバーの証明書や kubeconfig に IP ではなく名前（cp）が埋め込まれ、VM の IP が変わっても /etc/hosts の修正だけで済みます（01 の「IP についての注意」を参照）。使う場合は次のように両方を指定します。

```bash
# sudo kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint=cp:6443
```

## 2. kubectl の設定

kubeadm が生成する管理者用の kubeconfig（/etc/kubernetes/admin.conf）は root しか読めないため、一般ユーザーのホームにコピーして所有者を変えておくことで、sudo なしで kubectl を使えるようにします。

```bash
# ubuntu@cp
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 3. join コマンドの控え

kubeadm init の出力の最後に、worker をクラスタに参加させるための kubeadm join コマンドが表示されます。token は worker が API サーバーに認証するため、discovery-token-ca-cert-hash は worker が接続先の API サーバーを本物だと検証するために使われます。後で worker で実行するので控えておきます。

```bash
kubeadm join xxx:xxx --token xxx \
    --discovery-token-ca-cert-hash sha256:xxx
```

token の有効期限は 24 時間のため、期限が切れた場合や控え忘れた場合は cp で作り直します。

```bash
# sudo kubeadm token create --print-join-command
```

## 4. 既存の CNI 設定の確認

CRI-O はインストール時に bridge の CNI 設定を /etc/cni/net.d/ に置くことがあります。これが有効なままだと Cilium ではなく bridge で Pod にネットワークが割り当てられ、ノードをまたいだ Pod 通信ができなくなるため、事前に確認します。bridge の設定ファイルがすでに .disabled などで無効化されていれば何もする必要はありません。

```bash
ls /etc/cni/net.d/
```

有効な bridge の設定ファイルが残っている場合は、拡張子を変えて読み込まれないようにします（ファイル名は ls の結果に合わせてください）。

```bash
# sudo mv /etc/cni/net.d/11-crio-ipv4-bridge.conflist /etc/cni/net.d/11-crio-ipv4-bridge.conflist.disabled
```

## 5. Cilium Gateway API CRD の導入

Cilium の Gateway API 機能は、Gateway や HTTPRoute などのリソース定義（CRD）がクラスタに存在しないと有効になりません。CRD は Cilium には含まれていないため、Cilium をインストールする前に Gateway API 本体の CRD を入れておきます。

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_backendtlspolicies.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_tlsroutes.yaml
```

## 6. Cilium CLI のインストール

kubeadm は Pod 間通信の仕組み（CNI プラグイン）を含んでおらず、CNI が入るまでノードは NotReady、CoreDNS は Pending のままになります。CNI として Cilium を使い、kube-proxy の代わりと Gateway API も Cilium に任せるため、まずインストールや状態確認を行う cilium CLI を入れます。

最新の安定版を取得し、VM の CPU アーキテクチャ（Apple Silicon 上の Multipass なら arm64）に合ったバイナリを選びます。ダウンロードしたファイルが壊れていたり改ざんされていないことを sha256sum で検証してから展開します。

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## 7. Cilium のインストール

通常 Pod は kubernetes Service の ClusterIP 経由で API サーバーに接続しますが、kube-proxy を入れていないため、Cilium が起動するまでは ClusterIP への通信が転送されません。そのため、Cilium には API サーバーの実際のアドレスとポートを直接指定する必要があります。kubeconfig に書かれている接続先を確認します。

```bash
kubectl config view --minify | grep server
# server: https://xxx.xxx.xxx.x:6443
```

cilium CLI が ~/.kube/config を使ってクラスタに Cilium の DaemonSet や Operator をデプロイします。バージョンを指定しておくことで、手順を再実行したときも同じ構成を再現できます。

- kubeProxyReplacement=true：kube-proxy の代わりに Cilium が Service の通信を振り分けます
- k8sServiceHost / k8sServicePort：上で確認した API サーバーの IP とポートを指定します
- gatewayAPI.enabled=true：5 で入れた CRD を使い、Cilium を Gateway API のコントローラーとして動かします

```bash
cilium install --version x.xx.x \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API サーバーの IP> \
  --set k8sServicePort=6443 \
  --set gatewayAPI.enabled=true
```

## 8. インストール状況の確認

Cilium の Pod が起動しきる前に次の作業に進むと、ノードが NotReady のままだったり Pod 通信が失敗したりするため、--wait で準備完了まで待ちます。

```bash
cilium status --wait
```

CNI が入ったことで cp が Ready になり、Pending だった CoreDNS などが Running になっていることを確認します。

```bash
kubectl get nodes
kubectl get pods -A
```

## 9. worker の参加

worker1・worker2 をクラスタに参加させるため、3 で控えた kubeadm join コマンドを各 worker の VM 内で実行します。/etc/kubernetes 配下に設定ファイルを書き込むため root 権限が必要です。

```bash
# worker1, worker2
# sudo kubeadm join xxx:xxx --token xxx \
#     --discovery-token-ca-cert-hash sha256:xxx
```

worker が参加すると Cilium の DaemonSet が自動で worker にも Pod を配置します。その完了までは NotReady と表示されるため、cp から全ノードが Ready になるまで確認します。

```bash
# cp
kubectl get nodes
```

## 10. 疎通テスト

ノードが Ready でも、ノードをまたいだ Pod 間通信や Service、DNS が実際に機能しているとは限らないため、Cilium の接続テストでクラスタのネットワークが正しく動いていることを確かめます。

```bash
cilium connectivity test
```

## 11. 参考サイト

[Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
[Prepare Cilium Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
[Installing Cilium CNI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#create-the-cluster)
