# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/containerd/cri-containerd/releases/download/v1.0.0-beta.1/cri-containerd-1.0.0-beta.1.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.2/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.2/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.2/bin/linux/amd64/kubelet

for instance in 10.0.10.5 10.0.10.6 10.0.10.7; do
  scp cni-plugins-amd64-v0.6.0.tgz cri-containerd-1.0.0-beta.1.linux-amd64.tar.gz  kubectl kube-proxy kubelet ubuntu@${instance}:~/
done

```
The commands in this lab must be run on each worker instance: `worker-0`, `worker-1`, and `worker-2`. Login to each worker instance using the `gcloud` command. Example:

```
gcloud compute ssh worker-0
```

## Provisioning a Kubernetes Worker Node



Create the installation directories:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
```

```
sudo tar -xvf cri-containerd-1.0.0-beta.1.linux-amd64.tar.gz -C /
```

```
chmod +x kubectl kube-proxy kubelet
```

```
sudo mv kubectl kube-proxy kubelet /usr/local/bin/
```

### Configure CNI Networking

Retrieve the Pod CIDR range for the current compute instance:

```
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

Create the `bridge` network configuration file:

```
POD_CIDR=10.2.7.0/24
cat > 10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create the `loopback` network configuration file:

```
cat > 99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```

Move the network configuration files to the CNI configuration directory:

```
sudo mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

### Configure the Kubelet

```

sudo mv $(hostname)-key.pem $(hostname).pem /var/lib/kubelet/
```

```
sudo mv $(hostname).kubeconfig /var/lib/kubelet/kubeconfig
```

```
sudo mv ca.pem /var/lib/kubernetes/
```

Create the `kubelet.service` systemd unit file:

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cluster-dns=10.0.11.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --pod-cidr=${POD_CIDR} \\
  --register-node=true \\
  --require-kubeconfig \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/$(hostname).pem \\
  --tls-private-key-file=/var/lib/kubelet/$(hostname)-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy.service` systemd unit file:

```
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.2.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```
sudo mv kubelet.service kube-proxy.service /etc/systemd/system/

sudo mv kubelet.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable containerd cri-containerd kubelet kube-proxy
```

```
sudo systemctl start containerd cri-containerd kubelet kube-proxy

ubuntu@vm10-0-10-5:/etc/systemd/system$ service kubelet status
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-01-26 00:48:40 CST; 18s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 24285 (kubelet)
    Tasks: 10
   Memory: 23.7M
      CPU: 374ms
   CGroup: /system.slice/kubelet.service
           └─24285 /usr/local/bin/kubelet --allow-privileged=true --anonymous-auth=false --authorization-mode=Webhook --client-ca-file=/var/lib/kub

Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.096327   24285 factory.go:86] Registering Raw factory
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.096489   24285 manager.go:1178] Started watching for new ooms in manager
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.097377   24285 manager.go:329] Starting recovery of all containers
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.155245   24285 manager.go:334] Recovery completed
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.185065   24285 kubelet_node_status.go:273] Setting node annotation to enable vol
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.186791   24285 kubelet_node_status.go:431] Recording NodeHasSufficientDisk event
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.187031   24285 kubelet_node_status.go:431] Recording NodeHasSufficientMemory eve
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.187246   24285 kubelet_node_status.go:431] Recording NodeHasNoDiskPressure event
Jan 26 00:48:41 vm10-0-10-5.ksc.com kubelet[24285]: I0126 00:48:41.187463   24285 kubelet_node_status.go:82] Attempting to register node vm10-0-10-
Jan 26 00:48:42 vm10-0-10-5.ksc.com systemd[1]: Started Kubernetes Kubelet.
```

###启动报错
```
ubuntu@vm10-0-10-6:/etc/systemd/system$ ls
ctrl-alt-del.target     iscsi.service                paths.target.wants     sysinit.target.wants
default.target.wants    kubelet.service.d            shutdown.target.wants  syslog.service
getty.target.wants      multi-user.target.wants      sockets.target.wants   timers.target.wants
graphical.target.wants  network-online.target.wants  sshd.service
ubuntu@vm10-0-10-6:/etc/systemd/system$ sudo rm -fr kubelet.service.d 
[sudo] password for ubuntu: 
ubuntu@vm10-0-10-6:/etc/systemd/system$ 
```

```
 kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-01-26 00:55:01 CST; 1min 20s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 9185 (kubelet)
    Tasks: 10
   Memory: 23.6M
      CPU: 859ms
   CGroup: /system.slice/kubelet.service
           └─9185 /usr/local/bin/kubelet --allow-privileged=true --anonymous-auth=false --authorization-mode=Webhook --client-ca-file=/var/lib/kubernetes/ca.pem --cluster-dns=10.3.0.10 --cluster-domain=cluster.local --container-runtime=remote --container-runtime-endpoint=unix:///var/run/cri-containerd.sock --image-pull-progress-deadline=2m --kubeconfig=/var/lib/kubelet/kubeconfig --network-plugin=cni --pod-cidr=10.2.6.0/24 --register-node=true --require-kubeconfig --runtime-request-timeout=15m --tls-cert-file=/var/lib/kubelet/10.0.10.6.pem --tls-private-key-file=/var/lib/kubelet/10.0.10.6-key.pem --v=2

Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: E0126 00:56:02.314976    9185 kubelet_node_status.go:106] Unable to register node "vm10-0-10-6.ksc.com" with API server: Post https://120.131.1.200:6443/api/v1/nodes: dial tcp 120.131.1.200:6443: i/o timeout
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: I0126 00:56:02.716069    9185 kubelet_node_status.go:273] Setting node annotation to enable volume controller attach/detach
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: I0126 00:56:02.719723    9185 kubelet_node_status.go:431] Recording NodeHasSufficientDisk event message for node vm10-0-10-6.ksc.com
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: I0126 00:56:02.719781    9185 kubelet_node_status.go:431] Recording NodeHasSufficientMemory event message for node vm10-0-10-6.ksc.com
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: I0126 00:56:02.719923    9185 kubelet_node_status.go:431] Recording NodeHasNoDiskPressure event message for node vm10-0-10-6.ksc.com
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: I0126 00:56:02.719954    9185 kubelet_node_status.go:82] Attempting to register node vm10-0-10-6.ksc.com
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: E0126 00:56:02.966067    9185 reflector.go:205] k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:47: Failed to list *v1.Pod: Get https://120.131.1.200:6443/api/v1/pods?fieldSelector=spec.nodeName%3Dvm10-0-10-6.ksc.com&limit=500&resourceVersion=0: dial tcp 120.131.1.200:6443: i/o timeout
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: E0126 00:56:02.966849    9185 reflector.go:205] k8s.io/kubernetes/pkg/kubelet/kubelet.go:465: Failed to list *v1.Service: Get https://120.131.1.200:6443/api/v1/services?limit=500&resourceVersion=0: dial tcp 120.131.1.200:6443: i/o timeout
Jan 26 00:56:02 vm10-0-10-6.ksc.com kubelet[9185]: E0126 00:56:02.968055    9185 reflector.go:205] k8s.io/kubernetes/pkg/kubelet/kubelet.go:474: Failed to list *v1.Node: Get https://120.131.1.200:6443/api/v1/nodes?fieldSelector=metadata.name%3Dvm10-0-10-6.ksc.com&limit=500&resourceVersion=0: dial tcp 120.131.1.200:6443: i/o timeout
Jan 26 00:56:09 vm10-0-10-6.ksc.com kubelet[9185]: E0126 00:56:09.720087    9185 event.go:209] Unable to write event: 'Post https://120.131.1.200:6443/api/v1/namespaces/default/events: dial tcp 120.131.1.200:6443: i/o timeout' (may retry after sleeping)
~                                                                                             
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## Verification

```
ubuntu@vm10-0-10-2:~$ kubectl get nodes
NAME                  STATUS    ROLES     AGE       VERSION
vm10-0-10-6.ksc.com   Ready     <none>    31s       v1.9.2
```

Login to one of the controller nodes:

```
gcloud compute ssh controller-0
```

List the registered Kubernetes nodes:

```
kubectl get nodes
```

> output

```
ubuntu@vm10-0-10-2:~$ kubectl get nodes
NAME                  STATUS    ROLES     AGE       VERSION
vm10-0-10-5.ksc.com   Ready     <none>    21s       v1.9.2
vm10-0-10-6.ksc.com   Ready     <none>    15m       v1.9.2
vm10-0-10-7.ksc.com   Ready     <none>    8m        v1.9.2
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
