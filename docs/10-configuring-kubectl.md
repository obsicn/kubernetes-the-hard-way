# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes-the-hard-way` static IP address:

```
ubuntu@vm10-0-10-200:~/ca$ kubectl get node
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

```
ubuntu@vm10-0-10-200:~$ ls
admin-key.pem      admin.pem      ca.pem

```

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
KUBERNETES_PUBLIC_ADDRESS=120.131.1.200
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
```

```
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem
```

```
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

```
kubectl config use-context kubernetes-the-hard-way
```

## Verification

Check the health of the remote Kubernetes cluster:

```
kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
ubuntu@vm10-0-10-200:~$ kubectl get node
NAME                  STATUS    ROLES     AGE       VERSION
vm10-0-10-5.ksc.com   Ready     <none>    5m        v1.9.2
vm10-0-10-6.ksc.com   Ready     <none>    20m       v1.9.2
vm10-0-10-7.ksc.com   Ready     <none>    13m       v1.9.2
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
