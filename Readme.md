# Kubernetes Workshop Centos 7

Kubernetes is a system for managing containerized applications in a clustered environment.
It provides basic mechanisms for deployment, maintenance and scaling of applications on public, private or hybrid setups.
It also comes with self-healing features where containers can be auto provisioned, restarted or even replicated.

* Kubernetes Components

Kubernetes works in server-client setup, where it has a master providing centralized control for a number of minions.
We will be deploying a Kubernetes master with three minions, as illustrated in the diagram further below.

Kubernetes has several components:

```
etcd - A highly available key-value store for shared configuration and service discovery.
flannel - An etcd backed network fabric for containers.
kube-apiserver - Provides the API for Kubernetes orchestration.
kube-controller-manager - Enforces Kubernetes services.
kube-scheduler - Schedules containers on hosts.
kubelet - Processes a container manifest so the containers are launched according to how they are described.
kube-proxy - Provides network proxy services.
```

## Deployment on CentOS 7

We will need 4 servers, running on CentOS 7.1 64 bit with minimal install.
All components are available directly from the CentOS extras repository which is enabled by default.
The following architecture diagram illustrates where the Kubernetes components should reside:

* Prerequisites


1.Disable iptables on each node to avoid conflicts with Docker iptables rules:


```
$ systemctl stop firewalld
$ systemctl disable firewalld
```


2.Install NTP and make sure it is enabled and running:

```
$ yum -y install ntp
$ systemctl start ntpd
$ systemctl enable ntpd
```

3.Install some tools

```
yum -y install git net-tools telnet vim
```

4.Get the git files
This contains all the necessary scripts and additional information.
We are probably not going to use these but it provides a reference.

```
mkdir /git
cd /git
git clone https://github.com/stackbuilder/kubernetes-workshop
````

## The following steps should be performed on the master.

1.Install etcd, Kubernetes and flannel through yum:

```
$ yum -y install etcd kubernetes flannel
```

2.Configure etcd to listen to all IP addresses inside /etc/etcd/etcd.conf.
Ensure the following lines are uncommented, and assign the following values:

```
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
```

3.Configure Kubernetes API server inside /etc/kubernetes/apiserver.
Ensure the following lines are uncommented, and assign the following values:

```
KUBE_API_ADDRESS="--address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBELET_PORT="--kubelet_port=10250"
KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_API_ARGS=""
```

4.Start and enable etcd, kube-apiserver, kube-controller-manager and kube-scheduler:

```
$ for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler  kube-proxy flanneld; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```


5.Define flannel network configuration in etcd. This configuration will be pulled by flannel service on minions:

```
$ etcdctl mk /coreos.com/network/config '{"Network":"172.17.0.0/16"}'
```

6.At this point, we should notice that nodes' status returns nothing because we haven’t started any of them yet:

```
$ kubectl get nodes
NAME             LABELS              STATUS
```

## Setting up Kubernetes Minions (Nodes)

The following steps should be performed on minion1 and  minion2  unless specified otherwise.

1.Install flannel and Kubernetes using yum.

```
$ yum -y install flannel kubernetes
```

2.Configure etcd server for flannel service.

Update the following line inside /etc/sysconfig/flanneld to connect to the respective master.

```
FLANNEL_ETCD="http://master:2379"

```

3.Configure Kubernetes default config at /etc/kubernetes/config.
Ensure you update the KUBE_MASTER value to connect to the Kubernetes master API server.


```
KUBE_MASTER="--master=http://master:8080"
```

4.Configure kubelet service inside /etc/kubernetes/kubelet as below:

minion1:

```
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
# change the hostname to this host’s IP address
KUBELET_API_SERVER="--api_servers=http://master:8080"
KUBELET_ARGS=""
```

minion2:

```
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
# change the hostname to this host’s IP address
KUBELET_API_SERVER="--api_servers=http://master:8080"
KUBELET_ARGS=""
```

5.Start and enable kube-proxy, kubelet, docker and flanneld services:

```
$ for SERVICES in kube-proxy kubelet docker flanneld; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

6.On each minion, you should notice that you will have two new interfaces added, docker0 and flannel0.
You should get different range of IP addresses on flannel0 interface on each minion, similar to below:

minion1:
```
$ ip a | grep flannel | grep inet
inet 172.17.45.0/16 scope global flannel0
```

minion2:
```
$ ip a | grep flannel | grep inet
inet 172.17.38.0/16 scope global flannel0
```

7.Now login to Kubernetes master node and verify the minions’ status:

```
$ kubectl get nodes
NAME             LABELS                                  STATUS
192.168.50.131   kubernetes.io/hostname=192.168.50.131   Ready
192.168.50.132   kubernetes.io/hostname=192.168.50.132   Ready
192.168.50.133   kubernetes.io/hostname=192.168.50.133   Ready
```

You are now set. The Kubernetes cluster is now configured and running. We can start to play around with pods.


## Creating Pods (Containers)

To create a pod, we need to define a yaml file in the Kubernetes master, and use the kubectl command to create it based on the definition.
