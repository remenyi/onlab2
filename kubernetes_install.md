# Kubernetes install

## Container runtime telepítés dockerrel

Dependency-k:

```bash
 sudo apt-get update

 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

Docker repo gpg kulcs:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Repo hozzáadása:

```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

Installálás:

```bash
 sudo apt-get update

 sudo apt-get install docker-ce docker-ce-cli containerd.io

```

A Docker használja a systemd-t a cgroup-ok menedzseléséhez:

```bash
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Docker újraindítása:

```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Kubeadm, kubelet és kubectl telepítése

Dependency-k installálása:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Google Cloud kulcsok:

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Kubernetes repo hozzáadaása:

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Kubeadm, kubelet és kubectl telepítése

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Klaszter elkészítése

Csak a contol-plane node-on:

```bash
kubeadm init --pod-network-cidr <hálózat>  --apiserver-advertise-address <belső hálózati interfész címe>
export KUBECONFIG=/etc/kubernetes/admin.conf
```

A pod hálózatnak érdemes megadni a 10.244.0.0/16-os hálózatot, mert a flannel alapértelmezetten ezt használja. Elvileg a --pod-network-cidr flag-ben megadott hálózatot kéne használnia, de nem teszi.

Hálózat hozzáadása ugyancsak a control-plane node-okon:

```bash
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

A többi node hozzáadása a control-plane node-on a `kubeadm init` parancs kiadása után megjelenő utasítások alapján:

*A tmuxban másolás módba a C-b [ kombinációval lehet eljutni, majd a C-space kombinációval lehet a kijelölést elkezdeni. A kijelölést a C-b C-w-vel lehet befejezni, majd a kijelölt szöveget a C-b ] kombinációval lehet beilleszteni*

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

Ha nem működne valamiért:

```bash
kubeadm reset
rm -r /etc/cni/net.d
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

A contol-plane node-on a podok futtatásának engedélyezése:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Egy új node hozzáadása miután lejárt a hash

```bash
# A masterről kinyerjük az ip-t, a tokent és a token hasht
TOKEN=$(kubeadm token generate)
kubeadm token create $TOKEN --print-join-command --ttl=0
# A workeren kiadjuk a masteren kapott kubeadm join parancsot
```