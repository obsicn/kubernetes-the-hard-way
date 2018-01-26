# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the `encryption-config.yaml` encryption config file to each controller instance:

```
for instance in 10.0.10.2 10.0.10.3 10.0.10.4; do
  scp encryption-config.yaml ubuntu@${instance}:~/
done
```
master节点上的文件：
```
ubuntu@vm10-0-10-2:~$ ls 
ca-key.pem  ca.pem  encryption-config.yaml  kubernetes-key.pem  kubernetes.pem
```

Node上的文件

```
ubuntu@vm10-0-10-5:~$ ls 
10.0.10.5-key.pem  10.0.10.5.kubeconfig  10.0.10.5.pem  ca.pem  kube-proxy.kubeconfig
```
Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
