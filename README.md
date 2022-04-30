### Notes of how I created a two node Kubernetes cluster that is using cri-o runtime in my home lab.

```
[+] Software , IP info

192.168.1.80/24 controlplane
192.168.1.81/24 node01


root@controlplane:~# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.4 LTS
Release:        20.04
Codename:       focal
root@controlplane:~#



[+] Edit the /etc/hosts, /etc/hostname. Do a ping test between hosts using hostnames make sure it works.

[+] make sure you can do a curl/wget test to google.com :) (fix any DNS issues)

[+] Fix any NTP issues.

On Both Master & worker node (as root)
-----------------------------------------
[+] Add below on each machine

root@controlplane:~#vi /etc/ssh/sshd_config
PasswordAuthentication yes
PermitRootLogin yes
root@controlplane:~#reboot

[+] Continue .... on both Master & worker node (as root)

sed -i '/swap/d' /etc/fstab
swapoff -a
systemctl disable --now ufw

cat >>/etc/modules-load.d/crio.conf<<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

OS=xUbuntu_20.04
VERSION=1.20

cat >>/etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list<<EOF
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF

cat >>/etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list<<EOF
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers-cri-o.gpg add -


apt update && apt install -qq -y cri-o cri-o-runc cri-tools
apt install runc

cat >>/etc/crio/crio.conf.d/02-cgroup-manager.conf<<EOF
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
EOF

Note :- Please realize that CRI-O is now configured to use "cgroupfs". Kubelet by default is always going to use systemd. 
        At the point of cluster creation (kubeadm init), you need to create a "kubeadm-config.yaml" and tell kubelet to start with "cgroupfs" as well.

[+] At this point CRI-O is installed and we can check by running below commands

root@controlplane:~# curl -s --unix-socket /var/run/crio/crio.sock http://localhost/info
{"storage_driver":"overlay","storage_root":"/var/lib/containers/storage","cgroup_driver":"cgroupfs","default_id_mappings":{"uids":[{"container_id":0,"host_id":0,"size":4294967295}],"gids":[{"container_id":0,"host_id":0,"size":4294967295}]}}
root@controlplane:~#

root@node01:~# curl -s --unix-socket /var/run/crio/crio.sock http://localhost/info
{"storage_driver":"overlay","storage_root":"/var/lib/containers/storage","cgroup_driver":"cgroupfs","default_id_mappings":{"uids":[{"container_id":0,"host_id":0,"size":4294967295}],"gids":[{"container_id":0,"host_id":0,"size":4294967295}]}}
root@node01:~#

[+] Install kubeadm , kubelet & kubectl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

apt install -qq -y kubeadm=1.22.0-00 kubelet=1.22.0-00 kubectl=1.22.0-00


[+] Create a file to tell kubelet to be configured to use cgroupfs
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
root@controlplane:~# cat kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.22.0
networking:
  podSubnet: "10.244.0.0/16"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: cgroupfs
root@controlplane:~#


[+] As I shall be using Flannel (ran into some challenges) change the ranges subnet to /24 from /16. Read more here:
https://stackoverflow.com/questions/62408028/kubelet-failed-to-createpodsandbox-for-coredns-failed-to-set-bridge-addr-c
root@controlplane:/etc/cni/net.d# cat 100-crio-bridge.conf | grep -i ranges -A 1
        "ranges": [
            [{ "subnet": "10.85.0.0/24" }],
root@controlplane:/etc/cni/net.d#
root@controlplane:/etc/cni/net.d#cd
root@controlplane:~# 
root@controlplane:~# 
root@controlplane:~# systemctl daemon-reload && systemctl restart  crio
root@controlplane:~# 
root@controlplane:~# 
root@controlplane:~# systemctl status crio | grep -i activ
     Active: active (running) since Sat 2022-04-30 14:01:00 PDT; 36min ago
root@controlplane:~#



Only on Master node Run the below command and capture output
-------------------------------------------------------------
root@controlplane:~# cat kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.22.0
networking:
  podSubnet: "10.244.0.0/16"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: cgroupfs
root@controlplane:~#
root@controlplane:~# kubeadm init --config kubeadm-config.yaml
<snip>

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.80:6443 --token <snip> \
        --discovery-token-ca-cert-hash <snip>
root@controlplane:~#


Note :- Verify that file /var/lib/kubelet/config.yaml used by kubelet, is created by kubeadm at cluster creation time & indeed has correctly picked up cgroupfs.
root@controlplane:~# 
root@controlplane:~# systemctl show --property=Environment kubelet
Environment=[unprintable] KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml
root@controlplane:~# 
root@controlplane:~# cat /var/lib/kubelet/config.yaml | grep -i cgroup
cgroupDriver: cgroupfs
root@controlplane:~#


[+] Created a Flannel CNI

root@controlplane:~# kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
root@controlplane:~#

root@controlplane:~# kubectl get nodes -o wide
NAME           STATUS   ROLES                  AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
controlplane   Ready    control-plane,master   16m    v1.22.0   192.168.1.80   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   cri-o://1.20.7
node01         Ready    <none>                 8m5s   v1.22.0   192.168.1.81   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   cri-o://1.20.7
root@controlplane:~#


Woot Woot !!!

[+] On Worker

root@node01:~# kubeadm join 192.168.1.80:6443 --token <snip> \
>         --discovery-token-ca-cert-hash <snip>
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

root@node01:~#


[+] Just in case you need the token for a new worker
root@controlplane:~# kubeadm token create --print-join-command
kubeadm join 192.168.1.80:6443 --token <snip> --discovery-token-ca-cert-hash <snip>
root@controlplane:~#

[+] Check on controlplane all pods in kube-system
root@controlplane:~# kubectl get pods -n kube-system -o wide
NAME                                   READY   STATUS    RESTARTS        AGE   IP             NODE           NOMINATED NODE   READINESS GATES
coredns-78fcd69978-cfp2h               1/1     Running   0               27m   10.85.0.3      controlplane   <none>           <none>
coredns-78fcd69978-h5lhx               1/1     Running   0               27m   10.85.0.2      controlplane   <none>           <none>
etcd-controlplane                      1/1     Running   1               27m   192.168.1.80   controlplane   <none>           <none>
kube-apiserver-controlplane            1/1     Running   0               27m   192.168.1.80   controlplane   <none>           <none>
kube-controller-manager-controlplane   1/1     Running   1 (6m31s ago)   27m   192.168.1.80   controlplane   <none>           <none>
kube-flannel-ds-njx55                  1/1     Running   0               21m   192.168.1.80   controlplane   <none>           <none>
kube-flannel-ds-p2zjb                  1/1     Running   0               20m   192.168.1.81   node01         <none>           <none>
kube-proxy-bzxpk                       1/1     Running   0               27m   192.168.1.80   controlplane   <none>           <none>
kube-proxy-jkncn                       1/1     Running   0               20m   192.168.1.81   node01         <none>           <none>
kube-scheduler-controlplane            1/1     Running   1 (6m36s ago)   27m   192.168.1.80   controlplane   <none>           <none>
root@controlplane:~# 

[+] cri-o does not allow ping from a pod. To enable this in the file :-

root@controlplane:~# cat /etc/crio/crio.conf | grep -i default_capabilities -A 15
default_capabilities = [
        "CHOWN",
        "DAC_OVERRIDE",
        "FSETID",
        "FOWNER",
        "SETGID",
        "SETUID",
        "SETPCAP",
        "NET_BIND_SERVICE",
        "KILL",
]

# List of default sysctls. If it is empty or commented out, only the sysctls
# defined in the container json file by the user/kube will be added.
default_sysctls = [
]
root@controlplane:~#

Add "NET_RAW" 

root@controlplane:~# cat /etc/crio/crio.conf | grep -i default_capabilities -A 15
default_capabilities = [
        "CHOWN",
        "DAC_OVERRIDE",
        "FSETID",
        "FOWNER",
        "SETGID",
        "SETUID",
        "SETPCAP",
        "NET_BIND_SERVICE",
        "KILL",
        "NET_RAW",
]

# List of default sysctls. If it is empty or commented out, only the sysctls
# defined in the container json file by the user/kube will be added.
default_sysctls = [
root@controlplane:~#
root@controlplane:~# systemctl daemon-reload && systemctl restart  crio
root@controlplane:~#
root@controlplane:~# systemctl status crio | grep -i activ
     Active: active (running) since Sat 2022-04-30 15:32:56 PDT; 1min 8s ago
root@controlplane:~#

[+] For testing create two pods & do pings.
root@controlplane:~# kubectl run multitool-one --image=praqma/network-multitool
pod/multitool-one created
root@controlplane:~# kubectl run multitool-two --image=praqma/network-multitool
pod/multitool-two created
root@controlplane:~#
root@controlplane:~#
root@controlplane:~# kubectl get pods -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
multitool-one   1/1     Running   0          17s   10.244.1.5   node01   <none>           <none>
multitool-two   1/1     Running   0          10s   10.244.1.6   node01   <none>           <none>
root@controlplane:~#
root@controlplane:~#
root@controlplane:~# kubectl exec -it multitool-one bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
bash-5.1#
bash-5.1# ping 10.244.1.6
PING 10.244.1.6 (10.244.1.6) 56(84) bytes of data.
64 bytes from 10.244.1.6: icmp_seq=1 ttl=64 time=0.191 ms
64 bytes from 10.244.1.6: icmp_seq=2 ttl=64 time=0.081 ms
^C
--- 10.244.1.6 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1016ms
rtt min/avg/max/mdev = 0.081/0.136/0.191/0.055 ms
bash-5.1# exit
exit
root@controlplane:~#


```
