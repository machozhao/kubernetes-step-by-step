# kubernetes-step-by-step
## 准备
### 准备好不少于3个的CentOS 7的VM
其中一个服务器为master
另外两个服务器为minion，即node
### 每个VM按照如下方式配置主机名和IP地址
#### /etc/hosts
```
192.168.42.20	k8smaster.example.com	k8smaster
192.168.42.21	k8snode1.example.com	k8snode1
192.168.42.22	k8snode2.example.com	k8snode2
```
#### set hostname
```
# For Master
hostnamectl set-hostname k8smaster
# For Node1
hostnamectl set-hostname k8snode1
# For Node2
hostnamectl set-hostname k8snode2
```
### 确保各个服务器间能够正常ping通
### 为了避免一些不必要的冲突，可以暂时关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```
## Master环境的安装
1. 安装etcd和kubernetes-master
```
yum -y install etcd kubernetes-master
```
1. 配置/etc/etcd/etcd.conf文件
```
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_LISTEN_PEER_URLS="http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
#ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
#
#[proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"
#
#[security]
#ETCD_CERT_FILE=""
#ETCD_KEY_FILE=""
#ETCD_CLIENT_CERT_AUTH="false"
#ETCD_TRUSTED_CA_FILE=""
#ETCD_AUTO_TLS="false"
#ETCD_PEER_CERT_FILE=""
#ETCD_PEER_KEY_FILE=""
#ETCD_PEER_CLIENT_CERT_AUTH="false"
#ETCD_PEER_TRUSTED_CA_FILE=""
#ETCD_PEER_AUTO_TLS="false"
#
#[logging]
#ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""
```
1. 配置/etc/kubernetes/apiserver文件
```
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://127.0.0.1:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"

# Add your own!
KUBE_API_ARGS=""
```
1. 启动etcd、kube-apiserver、kube-controller-manager、kube-scheduler等服务，设置为开机自动启动
```
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; 
    do systemctl restart $SERVICES; systemctl enable $SERVICES; systemctl status $SERVICES ; 
done
```
1. 配置flannel网络基本参数
```
etcdctl mk /atomic.io/network/config '{"Network":"172.17.0.0/16"}'
```

## 配置各个Kubernetes Node
1. 安装flannel和kubernetes-node
```
yum -y install flannel kubernetes-node
```
2. 配置flannel网络指定etcd服务，修改/etc/sysconfig/flanneld文件，内容如下：
```
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://k8smaster:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/atomic.io/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
```
3. 配置/etc/kubernetes/config文件
```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://k8smaster:8080"
```
4. 修改kubelet参数文件/etc/kubernetes/kubelet
```
###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k8snode1"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://k8smaster:8080"

# pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

# Add your own!
KUBELET_ARGS=""
```
5. 启动kube-proxy,kubelet,docker,flanneld等服务，并设置开机启动
```
for SERVICES in kube-proxy kubelet docker flanneld; do systemctl restart $SERVICES;systemctl enable $SERVICES;systemctl status $SERVICES; done
```

## 安装配置成功后，检查系统状态，在任何一个node或master上，运行如下命令：
```
kubectl get node
```
上述命令应该能够显示各个节点，并且其状态为Ready

## 常见问题
1. 运行kubectl create -f apache-pod.yaml时出现如下错误：
```
Error from server: error when creating "nginx-pod.yaml": Pod "nginx" is forbidden: no API token found for service account default/default, retry after the token is automatically created and added to the service account
```
解决方法如下：
修改master的/etc/kubernetes/apiserver文件的KUBE_ADMISSION_CONTROL相关的参数，去除其中的ServiceAccount选项：
```
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
```

## Reference
### 常见问题：http://www.cnblogs.com/ivictor/p/4998032.html
### Microservices demo: https://github.com/microservices-demo/microservices-demo
