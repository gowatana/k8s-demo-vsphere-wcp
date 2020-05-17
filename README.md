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
$ export PATH=$(pwd)/bin:$PATH
$ which kubectl
~/k8s-demo-vsphere-wcp/bin/kubectl
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"16+", GitVersion:"v1.16.7-2+bfe512e5ddaaaa", GitCommit:"bfe512e5ddaaaa7243d602d5d161fa09a57ecf3c", GitTreeState:"clean", BuildDate:"2020-03-03T03:40:35Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
$ kubectl-vsphere version
kubectl-vsphere: version 0.0.1, build 15839988, change 7857139
```

# WCP / vSphere Pod

## Login.

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

## create pod.

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

## Demo 1: Create Service and Deployment (Pods).

```
$ kubectl apply -f 101_demo-svc.yml
$ kubectl get svc -n lab-ns-01
NAME       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
demo-svc   LoadBalancer   10.96.0.60   192.168.70.34   80:30688/TCP   70s
$ kubectl get svc
NAME       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
demo-svc   LoadBalancer   10.96.0.60   192.168.70.34   80:30688/TCP   2m29s
```

```
$ kubectl apply -f 102_demo-dep.yml
deployment.apps/demo-deploy created
$ kubectl get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/demo-deploy-7f8745d7bb-b5xgj   1/1     Running   0          6m46s
pod/demo-deploy-7f8745d7bb-mxq5t   1/1     Running   0          6m46s

NAME               TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
service/demo-svc   LoadBalancer   10.96.0.60   192.168.70.34   80:30688/TCP   9m48s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo-deploy   2/2     2            2           6m46s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/demo-deploy-7f8745d7bb   2         2         2       6m46s
```

access demo-svc service.

```
$ kubectl get svc demo-svc | awk '{print $4}'
EXTERNAL-IP
192.168.70.34
$ kubectl get svc demo-svc --no-headers | awk '{print $4}'
192.168.70.34
$ DEMO_SVC_IP=$(kubectl get svc demo-svc --no-headers | awk '{print $4}')
$ echo $DEMO_SVC_IP
192.168.70.34
$ curl -s $DEMO_SVC_IP:80 | head -n 4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

delete demo service and deployment.

```
$ kubectl delete -f 101_demo-svc.yml -f 102_demo-dep.yml
service "demo-svc" deleted
deployment.apps "demo-deploy" deleted
```

## Demo 2: LB test.

create service and deployment.

```
$ kubectl apply -f 201_kuard-svc.yml
service/kuard-svc created
$ kubectl apply -f 202_kuard-dep.yml
deployment.apps/kuard created
```

get all.

```
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/kuard-59c44f7c97-588kw   1/1     Running   0          97s
pod/kuard-59c44f7c97-845ms   1/1     Running   0          97s
pod/kuard-59c44f7c97-mgf9m   1/1     Running   0          97s

NAME                TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
service/kuard-svc   LoadBalancer   10.96.0.128   192.168.70.35   80:30189/TCP   104s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kuard   3/3     3            3           98s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/kuard-59c44f7c97   3         3         3       98s
```

access kuard-svc from a Web browser.  
It should be a round robin of 3 nodes.

```
$ kubectl get svc kuard-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.35/
```

delete delete demo 2.

```
$ kubectl delete -f 201_kuard-svc.yml,202_kuard-dep.yml
service "kuard-svc" deleted
deployment.apps "kuard" deleted
```

## Demo 3: NetworkPolicy test.

WIP

## Demo 4: Cloud Native Storage (PVC) test.

get StorageClass.

```
$ kubectl get sc
NAME                        PROVISIONER              AGE
vm-storage-policy-nfs-vsk   csi.vsphere.vmware.com   3h44m
$ kubectl get sc vm-storage-policy-nfs-vsk -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  creationTimestamp: "2020-05-17T01:55:46Z"
  name: vm-storage-policy-nfs-vsk
  resourceVersion: "622230"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/vm-storage-policy-nfs-vsk
  uid: ab1cad9b-b17d-4fb8-8cb0-8fe0b788bd92
parameters:
  storagePolicyID: a77655d1-1f5a-4164-bf27-bddb69b25556
provisioner: csi.vsphere.vmware.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

create service.

```
$ kubectl apply -f 401_cns-demo-svc.yml
service/cns-demo-svc created
$ kubectl get svc
NAME           TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
cns-demo-svc   LoadBalancer   10.96.0.30   192.168.70.34   80:32655/TCP   13s
$ kubectl get svc cns-demo-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.34/
```

create pods (StatefulSet) without PV.

```
$ kubectl get all
NAME             READY   STATUS    RESTARTS   AGE
pod/nginx-ss-0   1/1     Running   0          67s
pod/nginx-ss-1   1/1     Running   0          51s
pod/nginx-ss-2   1/1     Running   0          36s

NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
service/cns-demo-svc   LoadBalancer   10.96.0.30   192.168.70.34   80:32655/TCP   2m58s

NAME                        READY   AGE
statefulset.apps/nginx-ss   3/3     67s
```

delete StatefulSet, in preparation for re-creation.

```
$ kubectl delete -f 402_demo-ss.yml
statefulset.apps "nginx-ss" deleted
```

create yaml.

```
$ sed "s/vsan-ftt-0/vm-storage-policy-nfs-vsk/" 403_demo-ss_with-cns.yml > demo-ss_with-cns.yml
$ diff 403_demo-ss_with-cns.yml demo-ss_with-cns.yml
31c31
<         volume.beta.kubernetes.io/storage-class: "vsan-ftt-0"
---
>         volume.beta.kubernetes.io/storage-class: "vm-storage-policy-nfs-vsk"
```

create StatefulSet with pvc.

```
$ kubectl apply -f demo-ss_with-cns.yml
statefulset.apps/nginx-ss created
$ kubectl get all
NAME             READY   STATUS    RESTARTS   AGE
pod/nginx-ss-0   1/1     Running   0          3m5s
pod/nginx-ss-1   1/1     Running   0          2m44s
pod/nginx-ss-2   1/1     Running   0          89s

NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
service/cns-demo-svc   LoadBalancer   10.96.0.30   192.168.70.34   80:32655/TCP   10m

NAME                        READY   AGE
statefulset.apps/nginx-ss   3/3     3m5s
```

get pvc.

```
$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
demo-pvc-nginx-ss-0   Bound    pvc-cb23dfef-028b-4fdb-9435-d1c20e283834   2Gi        RWO            vm-storage-policy-nfs-vsk   3m28s
demo-pvc-nginx-ss-1   Bound    pvc-22048084-554e-43e1-a23e-1ea4c511db5b   2Gi        RWO            vm-storage-policy-nfs-vsk   3m8s
demo-pvc-nginx-ss-2   Bound    pvc-2dcc144b-1608-4a39-8036-790d3ef7056d   2Gi        RWO            vm-storage-policy-nfs-vsk   113s
```

get pv.

```
$ kubectl get pv
Error from server (Forbidden): persistentvolumes is forbidden: User "sso:Administrator@vsphere.local" cannot list resource "persistentvolumes" in API group "" at the cluster scope
```

check vSphere Clint - Namespace menu.

delete StatefulSet.

```
$ kubectl delete -f demo-ss_with-cns.yml
statefulset.apps "nginx-ss" deleted
```

delete PVC.

```
$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
demo-pvc-nginx-ss-0   Bound    pvc-cb23dfef-028b-4fdb-9435-d1c20e283834   2Gi        RWO            vm-storage-policy-nfs-vsk   19m
demo-pvc-nginx-ss-1   Bound    pvc-22048084-554e-43e1-a23e-1ea4c511db5b   2Gi        RWO            vm-storage-policy-nfs-vsk   19m
demo-pvc-nginx-ss-2   Bound    pvc-2dcc144b-1608-4a39-8036-790d3ef7056d   2Gi        RWO            vm-storage-policy-nfs-vsk   18m
$ kubectl delete pvc --all
persistentvolumeclaim "demo-pvc-nginx-ss-0" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-1" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-2" deleted
$ kubectl get pvc
No resources found in lab-ns-01 namespace.
```

delete service.

```
$ kubectl delete svc cns-demo-svc
service "cns-demo-svc" deleted
```

# Create Tanzu Kubernetes Cluster

## set env

```
$ NS=lab-ns-02
```

## env check

get Namespace.

```
$ kubectl get ns
NAME                      STATUS   AGE
default                   Active   36h
kube-node-lease           Active   36h
kube-public               Active   36h
kube-system               Active   36h
lab-ns-01                 Active   26h
lab-ns-02                 Active   8m8s
vmware-system-capw        Active   36h
vmware-system-csi         Active   36h
vmware-system-kubeimage   Active   36h
vmware-system-nsx         Active   36h
vmware-system-registry    Active   36h
vmware-system-tkg         Active   36h
vmware-system-ucs         Active   36h
vmware-system-vmop        Active   36h
```

get VirtualMachineClasses

```
$ kubectl get virtualmachineclasses
NAME                 AGE
best-effort-large    36h
best-effort-medium   36h
best-effort-small    36h
best-effort-xlarge   36h
best-effort-xsmall   36h
guaranteed-large     36h
guaranteed-medium    36h
guaranteed-small     36h
guaranteed-xlarge    36h
guaranteed-xsmall    36h
```

get VM VirtualMachineImages from Contents library.

```
$ kubectl get virtualmachineimages
NAME                                                        AGE
ob-15957779-photon-3-k8s-v1.16.8---vmware.1-tkg.3.60d2ffd   22m
```

create yaml.

```
$ sed "s/vsan-ftt-0/vm-storage-policy-nfs-vsk/" 000_tkg-cluster-01.yml > tkg-cluster-01.yml
$ diff 000_tkg-cluster-01.yml tkg-cluster-01.yml
14c14
<       storageClass: vsan-ftt-0
---
>       storageClass: vm-storage-policy-nfs-vsk
18c18
<       storageClass: vsan-ftt-0
---
>       storageClass: vm-storage-policy-nfs-vsk
28,29c28,29
<       classes: ["vsan-ftt-0"]           # Named PVC storage classes
<       defaultClass: vsan-ftt-0          # Default PVC storage class
---
>       classes: ["vm-storage-policy-nfs-vsk"]           # Named PVC storage classes
>       defaultClass: vm-storage-policy-nfs-vsk          # Default PVC storage class
```

create tanzu kubernetes cluster.

```
$ kubectl -n $NS apply -f tkg-cluster-01.yml
tanzukubernetescluster.run.tanzu.vmware.com/tkg-cluster-01 created
$ kubectl get -n lab-ns-02 tanzukubernetesclusters.run.tanzu.vmware.com
NAME             CONTROL PLANE   WORKER   DISTRIBUTION                     AGE   PHASE
tkg-cluster-01   3               3        v1.16.8+vmware.1-tkg.3.60d2ffd   29m   running
```

login

```
$ kubectl vsphere login --server=$SCP_NODE --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace=$NS --tanzu-kubernetes-cluster-name=tkg-cluster-01

Username: administrator@vsphere.local
Password:
Logged in successfully.

You have access to the following contexts:
   192.168.70.33
   192.168.70.5
   lab-ns-01
   lab-ns-02
   tkg-cluster-01

If the context you wish to use is not in this list, you may need to try
logging in again later, or contact your cluster administrator.

To change context, use `kubectl config use-context <workload name>`
```

get current-context.

```
$ kubectl config current-context
tkg-cluster-01
$ kubectl config get-contexts tkg-cluster-01
CURRENT   NAME             CLUSTER         AUTHINFO                                        NAMESPACE
*         tkg-cluster-01   192.168.70.34   wcp:192.168.70.34:administrator@vsphere.local
$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE     VERSION
tkg-cluster-01-control-plane-hd6pf              Ready    master   2m2s    v1.16.8+vmware.1
tkg-cluster-01-control-plane-n9pj7              Ready    master   26m     v1.16.8+vmware.1
tkg-cluster-01-control-plane-tvbp7              Ready    master   5m21s   v1.16.8+vmware.1
tkg-cluster-01-workers-82dgq-7d6659ccd9-q44tw   Ready    <none>   106s    v1.16.8+vmware.1
tkg-cluster-01-workers-82dgq-7d6659ccd9-xbq9j   Ready    <none>   115s    v1.16.8+vmware.1
tkg-cluster-01-workers-82dgq-7d6659ccd9-zcqgp   Ready    <none>   12m     v1.16.8+vmware.1
```
## create demo Namespace.

```
$ kubectl create ns demo
namespace/demo created
```

create RoleBindig for PodSecurityPolicy.

```
$ kubectl -n demo apply -f 001_rolebinding_demo.yaml
rolebinding.rbac.authorization.k8s.io/rolebind-default-privileged-ns_demo_serviceaccounts created
```

## demo 1-B: 

create service and deployment.

```
$ kubectl -n demo apply -f 101_demo-svc.yml,102_demo-dep.yml
service/demo-svc created
deployment.apps/demo-deploy created
```

get all.

```$ kubectl -n demo get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/demo-deploy-7f8745d7bb-kbbfx   1/1     Running   0          66s
pod/demo-deploy-7f8745d7bb-q8kcj   1/1     Running   0          66s

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/demo-svc   LoadBalancer   198.61.33.231   192.168.70.36   80:32422/TCP   66s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo-deploy   2/2     2            2           66s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/demo-deploy-7f8745d7bb   2         2         2       67s
```

access service.

```
$ kubectl -n demo get svc demo-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.36/
$ curl http://192.168.70.36/
```

delete

```
$ kubectl -n demo delete -f 101_demo-svc.yml,102_demo-dep.yml
service "demo-svc" deleted
deployment.apps "demo-deploy" deleted
```

## demo 2-B: 

get current namespace.

```
$ kubectl config current-context
tkg-cluster-01
$ kubectl config get-contexts tkg-cluster-01
CURRENT   NAME             CLUSTER         AUTHINFO                                        NAMESPACE
*         tkg-cluster-01   192.168.70.34   wcp:192.168.70.34:administrator@vsphere.local
```

set namespace.

```
$ kubectl config set-context --current --namespace=demo
Context "tkg-cluster-01" modified.
$ kubectl config get-contexts tkg-cluster-01
CURRENT   NAME             CLUSTER         AUTHINFO                                        NAMESPACE
*         tkg-cluster-01   192.168.70.34   wcp:192.168.70.34:administrator@vsphere.local   demo
```

create service and deployment.

```
$ kubectl apply -f 201_kuard-svc.yml,202_kuard-dep.yml
service/kuard-svc created
deployment.apps/kuard created
$ kubectl -n demo get svc kuard-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.36/
```

get all.

```
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/kuard-59c44f7c97-76c86   1/1     Running   0          73s
pod/kuard-59c44f7c97-qdzv9   1/1     Running   0          73s
pod/kuard-59c44f7c97-wjj8f   1/1     Running   0          73s

NAME                TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
service/kuard-svc   LoadBalancer   198.56.86.16   192.168.70.36   80:31303/TCP   74s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kuard   3/3     3            3           73s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/kuard-59c44f7c97   3         3         3       73s
```

delete.

```
$ kubectl delete -f 201_kuard-svc.yml,202_kuard-dep.yml
service "kuard-svc" deleted
deployment.apps "kuard" deleted
```

## demo 3-B: 

WIP

## demo 4-B:

get StorageClass.

```
$ kubectl get sc vm-storage-policy-nfs-vsk
NAME                                  PROVISIONER              AGE
vm-storage-policy-nfs-vsk (default)   csi.vsphere.vmware.com   4h58m
$ kubectl get sc vm-storage-policy-nfs-vsk -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  creationTimestamp: "2020-05-17T09:14:31Z"
  labels:
    isSyncedFromSupervisor: "yes"
  name: vm-storage-policy-nfs-vsk
  resourceVersion: "71"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/vm-storage-policy-nfs-vsk
  uid: 5993a64d-f788-46ce-ad46-ed21896fd7f2
parameters:
  svStorageClass: vm-storage-policy-nfs-vsk
provisioner: csi.vsphere.vmware.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

create service.

```
$ kubectl apply -f 401_cns-demo-svc.yml
service/cns-demo-svc created
```

create StatefulSet.

```
$ kubectl apply -f 402_demo-ss.yml
statefulset.apps/nginx-ss created
$ kubectl get statefulset
NAME       READY   AGE
nginx-ss   3/3     5m58s
$ kubectl get all
NAME             READY   STATUS    RESTARTS   AGE
pod/nginx-ss-0   1/1     Running   0          6m18s
pod/nginx-ss-1   1/1     Running   0          6m10s
pod/nginx-ss-2   1/1     Running   0          5m15s

NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/cns-demo-svc   LoadBalancer   198.63.121.84   192.168.70.37   80:30112/TCP   6m28s

NAME                        READY   AGE
statefulset.apps/nginx-ss   3/3     6m18s
```

delete StatefulSet.

```
$ kubectl delete -f 402_demo-ss.yml
statefulset.apps "nginx-ss" deleted
```

create yaml.

```
$ sed "s/vsan-ftt-0/vm-storage-policy-nfs-vsk/" 403_demo-ss_with-cns.yml > demo-ss_with-cns_tanzu.yml
$ diff 403_demo-ss_with-cns.yml demo-ss_with-cns_tanzu.yml
31c31
<         volume.beta.kubernetes.io/storage-class: "vsan-ftt-0"
---
>         volume.beta.kubernetes.io/storage-class: "vm-storage-policy-nfs-vsk"
```

re-create StatefulSet. (with PVC)

```
$ kubectl apply -f demo-ss_with-cns_tanzu.yml
statefulset.apps/nginx-ss created
```

get all.

```
$ kubectl get all
NAME             READY   STATUS    RESTARTS   AGE
pod/nginx-ss-0   1/1     Running   0          2m32s
pod/nginx-ss-1   1/1     Running   0          102s
pod/nginx-ss-2   1/1     Running   0          40s

NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/cns-demo-svc   LoadBalancer   198.63.121.84   192.168.70.37   80:30112/TCP   11m

NAME                        READY   AGE
statefulset.apps/nginx-ss   3/3     2m33s
```

get pvc.

```
$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
demo-pvc-nginx-ss-0   Bound    pvc-9531a4ff-8230-4e36-96bc-c395ea64b91c   2Gi        RWO            vm-storage-policy-nfs-vsk   5m31s
demo-pvc-nginx-ss-1   Bound    pvc-25cd1ed4-c48b-4296-bd70-9bd05d92b388   2Gi        RWO            vm-storage-policy-nfs-vsk   4m41s
demo-pvc-nginx-ss-2   Bound    pvc-dcafc47b-1fdc-41cf-9b81-e76bce12f906   2Gi        RWO            vm-storage-policy-nfs-vsk   3m39s
```

get pv.

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS                REASON   AGE
pvc-25cd1ed4-c48b-4296-bd70-9bd05d92b388   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-1   vm-storage-policy-nfs-vsk            4m27s
pvc-9531a4ff-8230-4e36-96bc-c395ea64b91c   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-0   vm-storage-policy-nfs-vsk            5m40s
pvc-dcafc47b-1fdc-41cf-9b81-e76bce12f906   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-2   vm-storage-policy-nfs-vsk            3m52s
```

delete StatefulSet and PVC.

```
$ kubectl delete -f demo-ss_with-cns_tanzu.yml
statefulset.apps "nginx-ss" deleted
$ kubectl delete pvc --all
persistentvolumeclaim "demo-pvc-nginx-ss-0" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-1" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-2" deleted
```

EOF
