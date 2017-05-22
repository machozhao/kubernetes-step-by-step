### In all host and CentOS env, run
```
ARCH=x86_64
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-${ARCH}
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```

### Set proxy to access Google docker registry:
```
cd /etc/systemd/system/docker.service.d
```
创建文件http-proxy.conf， 内容如下：
```
[Service]
Environment="HTTP_PROXY=http://192.168.1.15:9666/"
Environment="HTTPS_PROXY=http://192.168.1.15:9666/"
Environment="NO_PROXY=localhost,127.0.0.1,localaddress,.localdomain.com"
```
之后运行如下命令重启docker
```
systemctl daemon-reload
systemctl restart docker
```
### Prepare Docker image, if you are in China
```bash
[root@master1 ~]# docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED             SIZE
gcr.io/google_containers/kube-proxy-amd64                v1.6.3              7d1bd9707c45        3 days ago          109.2 MB
gcr.io/google_containers/kube-apiserver-amd64            v1.6.3              b80d5b7319cc        3 days ago          150.6 MB
gcr.io/google_containers/kube-controller-manager-amd64   v1.6.3              d2888c09d1e6        3 days ago          132.8 MB
gcr.io/google_containers/kube-scheduler-amd64            v1.6.3              71a568bd21be        3 days ago          76.76 MB
quay.io/coreos/flannel                                   v0.7.1-amd64        cd4ae0be5e1b        3 weeks ago         77.76 MB
gcr.io/google_containers/k8s-dns-sidecar-amd64           1.14.1              fc5e302d8309        10 weeks ago        44.52 MB
gcr.io/google_containers/k8s-dns-kube-dns-amd64          1.14.1              f8363dbf447b        10 weeks ago        52.36 MB
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64     1.14.1              1091847716ec        10 weeks ago        44.84 MB
gcr.io/google_containers/etcd-amd64                      3.0.17              243830dae7dd        11 weeks ago        168.9 MB
gcr.io/google_containers/pause-amd64                     3.0                 99e59f495ffa        12 months ago       746.9 kB
```
### Init Master
```
kubeadm init --pod-network-cidr 10.244.0.0/16 
```
You will get these output
```
[root@master1 ~]# kubeadm init --pod-network-cidr 10.244.0.0/16 
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.6.3
[init] Using Authorization mode: RBAC
[preflight] Running pre-flight checks
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [master1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.42.60]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 1006.378389 seconds
[apiclient] Waiting for at least one node to register
[apiclient] First node has registered after 9.189422 seconds
[token] Using token: 39f97f.119eb214549d3170
[apiconfig] Created RBAC rules
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  sudo cp /etc/kubernetes/admin.conf $HOME/
  sudo chown $(id -u):$(id -g) $HOME/admin.conf
  export KUBECONFIG=$HOME/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 39f97f.119eb214549d3170 192.168.42.60:6443

```

### Init Flannel network
```
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Chech K8s status
```
[root@master1 ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.3", GitCommit:"0480917b552be33e2dba47386e51decb1a211df6", GitTreeState:"clean", BuildDate:"2017-05-10T15:48:59Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.3", GitCommit:"0480917b552be33e2dba47386e51decb1a211df6", GitTreeState:"clean", BuildDate:"2017-05-10T15:38:08Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"linux/amd64"}
[root@master1 ~]# 
[root@master1 ~]# kubectl get namespace
NAME          STATUS    AGE
default       Active    1h
kube-public   Active    1h
kube-system   Active    1h
[root@master1 ~]# 
[root@master1 ~]# kubectl get pod -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
etcd-master1                      1/1       Running   0          1h
kube-apiserver-master1            1/1       Running   0          1h
kube-controller-manager-master1   1/1       Running   0          1h
kube-dns-3913472980-nnn1t         3/3       Running   0          1h
kube-flannel-ds-1fghz             2/2       Running   0          1h
kube-proxy-0cwm9                  1/1       Running   0          1h
kube-scheduler-master1            1/1       Running   0          1h
[root@master1 ~]# 
[root@master1 ~]# kubectl get node
NAME      STATUS    AGE       VERSION
master1   Ready     1h        v1.6.3
[root@master1 ~]# 
[root@master1 ~]# 
[root@master1 ~]# kubectl get svc
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.96.0.1    <none>        443/TCP   1h
[root@master1 ~]# 
```

## Setup Kubernetes Node
### Copy admin config from master
```
scp root@master1:/root/admin.conf ~/
. ~/.bash_profile
```
### Apply addon for network
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### Registe as Node
```
kubeadm join --token 39f97f.119eb214549d3170 master1:6443
```
## Appendix#A Export Docker images:
```
docker save gcr.io/google_containers/kube-proxy-amd64:v1.6.3 > kube-proxy-amd64_v1.6.3.tar
docker save gcr.io/google_containers/kube-apiserver-amd64:v1.6.3 > kube-apiserver-amd64_v1.6.3.tar
docker save gcr.io/google_containers/kube-controller-manager-amd64:v1.6.3 > kube-controller-manager-amd64_v1.6.3.tar
docker save gcr.io/google_containers/kube-scheduler-amd64:v1.6.3 > kube-scheduler-amd64_v1.6.3.tar
docker save gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.1 > k8s-dns-sidecar-amd64_1.14.1.tar
docker save gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.1 > k8s-dns-kube-dns-amd64_1.14.1.tar
docker save gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.1 > k8s-dns-dnsmasq-nanny-amd64_1.14.1.tar
docker save gcr.io/google_containers/etcd-amd64:3.0.17 > etcd-amd64_3.0.17.tar
docker save gcr.io/google_containers/pause-amd64:3.0 > pause-amd64_3.0.tar
docker save quay.io/coreos/flannel:v0.7.1-amd64 > flannel_v0.7.1-amd64.tar
```
