# k8s-demo-vsphere-wcp

set env.

```
$ SCP_NODE=192.168.70.33
$ KUBECONFIG=./kubeconfig
```

Download kubectl.

```
$ curl -k -L -O https://${SCP_NODE}/wcp/plugin/linux-amd64/vsphere-plugin.zip
$ unzip vsphere-plugin.zip
Archive:  vsphere-plugin.zip
   creating: bin/
  inflating: bin/kubectl-vsphere
  inflating: bin/kubectl
```

set PATH

```
export PATH=$(pwd)/bin:$PATH
$ which kubectl
~/k8s-demo-vsphere-wcp/bin/kubectl
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"16+", GitVersion:"v1.16.7-2+bfe512e5ddaaaa", GitCommit:"bfe512e5ddaaaa7243d602d5d161fa09a57ecf3c", GitTreeState:"clean", BuildDate:"2020-03-03T03:40:35Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
$ kubectl-vsphere version
kubectl-vsphere: version 0.0.1, build 15839988, change 7857139
```

# WCP / vSphere Pod

Login.

```
$ kubectl vsphere login --server=$SCP_NODE --insecure-skip-tls-verify

Username: administrator@vsphere.local
Password:
Logged in successfully.

You have access to the following contexts:
   192.168.70.33
   192.168.70.5
   lab-ns-01

If the context you wish to use is not in this list, you may need to try
logging in again later, or contact your cluster administrator.

To change context, use `kubectl config use-context <workload name>`
```

change context

```
$ kubectl config use-context lab-ns-01
Switched to context "lab-ns-01".
```

Run Pod.

```
$ kubectl run -it --image=centos:7 --labels="app=jumpbox" --generator=run-pod/v1 --rm jbox
If you don't see a command prompt, try pressing enter.
[root@jbox /]#
[root@jbox /]# cat /etc/centos-release
CentOS Linux release 7.8.2003 (Core)
```

check nw connection, and show podvnic.

```
[root@jbox /]# yum install -y iproute
[root@jbox /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: podvnic: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 04:50:56:00:f8:04 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.210/28 brd 10.244.0.223 scope global podvnic
       valid_lft forever preferred_lft forever
```

get SC.

```
$ kubectl get StorageClass
NAME         PROVISIONER              AGE
vsan-ftt-0   csi.vsphere.vmware.com   17m
```

