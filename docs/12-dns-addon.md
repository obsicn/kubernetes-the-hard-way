# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

Deploy the `kube-dns` cluster add-on:

```

ubuntu@vm10-0-10-200:~$ kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
serviceaccount "kube-dns" created
configmap "kube-dns" created
deployment "kube-dns" created
The Service "kube-dns" is invalid: spec.clusterIP: Invalid value: "10.32.0.10": provided IP is not in the valid range. The range of valid IPs is 10.0.11.0/24

kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
```

> output

```
ubuntu@vm10-0-10-200:~$ git clone https://github.com/obsicn/kubernetes-the-hard-way
Cloning into 'kubernetes-the-hard-way'...
remote: Counting objects: 1155, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 1155 (delta 0), reused 2 (delta 0), pack-reused 1151
Receiving objects: 100% (1155/1155), 443.38 KiB | 170.00 KiB/s, done.
Resolving deltas: 100% (774/774), done.
Checking connectivity... done.
ubuntu@vm10-0-10-200:~$ vi kube
kube-apiserver           kubectl                  kubelet                  kubernetes-the-hard-way/ 
kube-controller-manager  kube-dns.yaml            kube-proxy               kube-scheduler           
ubuntu@vm10-0-10-200:~$ vi kubernetes-the-hard-way/
deployments/ docs/        .git/        .gitignore   LICENSE      README.md    
ubuntu@vm10-0-10-200:~$ vi kubernetes-the-hard-way/deployments/kube-dns.yaml 
ubuntu@vm10-0-10-200:~$ kubectl create -f kubernetes-the-hard-way/deployments/kube-dns.yaml 
service "kube-dns" created
Error from server (AlreadyExists): error when creating "kubernetes-the-hard-way/deployments/kube-dns.yaml": serviceaccounts "kube-dns" already exists
Error from server (AlreadyExists): error when creating "kubernetes-the-hard-way/deployments/kube-dns.yaml": configmaps "kube-dns" already exists
Error from server (AlreadyExists): error when creating "kubernetes-the-hard-way/deployments/kube-dns.yaml": deployments.extensions "kube-dns" already exists
ubuntu@vm10-0-10-200:~$ kubectl delete kube-dns
error: resource(s) were provided, but no name, label selector, or --all flag specified
ubuntu@vm10-0-10-200:~$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    2h
kube-public   Active    2h
kube-system   Active    2h
ubuntu@vm10-0-10-200:~$ kubectl delete kube-dns -n kube-system 
error: resource(s) were provided, but no name, label selector, or --all flag specified
ubuntu@vm10-0-10-200:~$ kubectl delete service kube-dns -n kube-system 
service "kube-dns" deleted
ubuntu@vm10-0-10-200:~$ kubectl delete serviceaccounts kube-dns -n kube-system 
serviceaccount "kube-dns" deleted
ubuntu@vm10-0-10-200:~$ kubectl delete configmaps kube-dns -n kube-system 
configmap "kube-dns" deleted
ubuntu@vm10-0-10-200:~$ kubectl delete deployments.extensions kube-dns -n kube-system 
deployment "kube-dns" deleted
ubuntu@vm10-0-10-200:~$ kubectl create -f kubernetes-the-hard-way/deployments/kube-dns.yaml 
serviceaccount "kube-dns" created
configmap "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created
```

List the pods created by the `kube-dns` deployment:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-3097350089-gq015   3/3       Running   0          20s
kube-dns-3097350089-q64qc   3/3       Running   0          20s
```

## Verification

Create a `busybox` deployment:

```
kubectl run busybox --image=busybox --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```
kubectl get pods -l run=busybox
```

> output

```
NAME                       READY     STATUS    RESTARTS   AGE
busybox-2125412808-mt2vb   1/1       Running   0          15s
```

Retrieve the full name of the `busybox` pod:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME -- bash
Error from server: error dialing backend: dial tcp: lookup vm10-0-10-7.ksc.com on 198.18.254.30:53: no such host

修改没台机器上的/etc/hosts文件。
127.0.0.1 localhost
10.0.10.200 vm10-0-10-200.ksc.com
10.0.10.2 vm10-0-10-2.ksc.com
10.0.10.3 vm10-0-10-3.ksc.com
10.0.10.4 vm10-0-10-4.ksc.com
10.0.10.5 vm10-0-10-5.ksc.com
10.0.10.6 vm10-0-10-6.ksc.com
10.0.10.7 vm10-0-10-7.ksc.com
~                               
ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME date




ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
Server:    10.3.0.10

ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME -- ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
ubuntu@vm10-0-10-200:~$ 


> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```


ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
Server:    10.3.0.10
Address 1: 10.3.0.10

nslookup: can't resolve 'kubernetes'
command terminated with exit code 1


ubuntu@vm10-0-10-200:~$ kubectl exec -ti $POD_NAME cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.3.0.10
options ndots:5



Next: [Smoke Test](13-smoke-test.md)
