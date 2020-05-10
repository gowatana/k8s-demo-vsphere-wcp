# k8s-demo-vsphere-wcp

Login.

```
$ SCP_NODE=172.22.18.65
$ kubectl vsphere login --server=$SCP_NODE --insecure-skip-tls-verify --vsphere-username=administrator@vsphere.local
```

change context

```
$ kubectl config use-context demo-1
Switched to context "demo-1".
```

Run Pod.

```
$ kubectl run -it --image=centos:7 --labels="app=jumpbox" --generator=run-pod/v1 --rm jbox
```

get SC.

```
$ kubectl get StorageClass
NAME         PROVISIONER              AGE
vsan-ftt-0   csi.vsphere.vmware.com   17m
```

