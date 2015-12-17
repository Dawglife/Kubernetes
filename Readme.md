Kubernetes is a system for managing containerized applications in a clustered environment.
It provides basic mechanisms for deployment, maintenance and scaling of applications on public, private or hybrid setups.
It also comes with self-healing features where containers can be auto provisioned, restarted or even replicated.

** Kubernetes Components **

Kubernetes works in server-client setup, where it has a master providing centralized control for a number of minions.
We will be deploying a Kubernetes master with three minions, as illustrated in the diagram further below.

Kubernetes has several components:

etcd - A highly available key-value store for shared configuration and service discovery.
flannel - An etcd backed network fabric for containers.
kube-apiserver - Provides the API for Kubernetes orchestration.
kube-controller-manager - Enforces Kubernetes services.
kube-scheduler - Schedules containers on hosts.
kubelet - Processes a container manifest so the containers are launched according to how they are described.
kube-proxy - Provides network proxy services.


Deployment on CentOS 7

We will need 4 servers, running on CentOS 7.1 64 bit with minimal install. All components are available directly from the CentOS extras repository which is enabled by default. The following architecture diagram illustrates where the Kubernetes components should reside:
