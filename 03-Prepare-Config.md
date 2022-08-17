# Generating Kubernetes Configuration

This part doesn't drift too much apart from the original guide. One of the changes is that I enforce the embedding of the certificates in the configuration file, as I'd like to be able to move those files around without worrying about the certificate reference, this could be a trade-off. 

Still inside the `pki` directory, common to next steps. 

For this step we need kubectl to be in the $PATH, move it to `/usr/local/bin` if it’s not already in a $PATH location. I found that by default `~/bin` is in the $PATH, however for the sake of consistency I always move it to `/usr/local/bin`. 

```shell
sudo mv ~/bin/kubectl /usr/local/bin/
```
```shell
KUBERNETES_PUBLIC_ADDRESS=192.168.5.150
```

## The `kubelet` Kubernetes Configuration File

```shell
for instance in p1 p2 p3; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done

Cluster "kubernetes-the-hard-way" set.
User "system:node:p1" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes-the-hard-way" set.
User "system:node:p2" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes-the-hard-way" set.
User "system:node:p3" set.
Context "default" created.
Switched to context "default".
```

## The `kube-proxy` Kubernetes Configuration File

```shell
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

Cluster "kubernetes-the-hard-way" set.
User "system:kube-proxy" set.
Context "default" created.
Switched to context "default".
```

## The `kube-controller-manager` Kubernetes Configuration File

```shell
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

Cluster "kubernetes-the-hard-way" set.
User "system:kube-controller-manager" set.
Context "default" created.
Switched to context "default".
```

## The `kube-scheduler` Kubernetes Configuration File

```shell
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

Cluster "kubernetes-the-hard-way" set.
User "system:kube-scheduler" set.
Context "default" created.
Switched to context "default".
```

## The `admin` Kubernetes Configuration File

```shell
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig

Cluster "kubernetes-the-hard-way" set.
User "admin" set.
Context "default" created.
Switched to context "default".
```

## Generating the Data Encryption Config and Key

```shell
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

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

## Distribute the Files

```shell
for instance in p1 p2 p3; do
  ssh ${instance} "mkdir config"
  scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/config/
done

p1.kubeconfig                                                         100% 6503     2.3MB/s   00:00    
kube-proxy.kubeconfig                                                 100% 6485     2.9MB/s   00:00    
p2.kubeconfig                                                         100% 6499     2.3MB/s   00:00    
kube-proxy.kubeconfig                                                 100% 6485     2.8MB/s   00:00    
p3.kubeconfig                                                         100% 6499     2.4MB/s   00:00    
kube-proxy.kubeconfig                                                 100% 6485     2.9MB/s   00:00 

cp \
  admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  encryption-config.yaml \
../config/
```
