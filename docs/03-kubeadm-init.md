
```bash
# cp
sudo kubeadm init
```

```bash
# ubuntu@cp
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

and then copy own my token to join control plane kubeadm like
```bash
kubeadm join xxx:xxx --token xxx \
    --discovery-token-ca-cert-hash sha256:xxx
```

If you already have a bridge.conflict in `/etc/cni/net.d/` directory, you need to move the bridge to bak or disabled. If the bridge is disabled, you have not to do anything so far.
```bash
ls /etc/cni/net.d/
```

Download Cilium using followed commands refered from (here)[https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#create-the-cluster].


```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
Install Cilium
```bash
cilium install x.xx.x
```

Check the progress of the installing
```bash
cilium status --wait
```

And also you can get the status with testing on running followed by commands.
```bash
kubectl get nodes
kubectl get pods -A
```

then you should join kubeadm for worker1, 2 while copied and pasted on the workers.
Once this is done, you can see the worker nodes on control plane that the workers are ready or pending.
```bash
# cp
kubectl get nodes
```

next, you can see testing of cilium running on the command.
```bash
cilium connectivity test
```


## x. 参考サイト
[Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
[Installing Cilium CNI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#create-the-cluster)
