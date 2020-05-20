# vSphere 7 with k8s DEMO

Superviser Cluster 有効化した環境を使ってみる。
* vSphere Pod のデモ →【WCP】
* Tanzu Kubernetes Cluster の作成
* Tanzu Kubernetes Cluster でのデモ →【TKC】

# 準備

kubectl のダウンロード

```
$ curl -k -L -O https://${SCP_NODE}/wcp/plugin/linux-amd64/vsphere-plugin.zip
$ unzip vsphere-plugin.zip
Archive:  vsphere-plugin.zip
   creating: bin/
  inflating: bin/kubectl-vsphere
  inflating: bin/kubectl
```

環境変数を調整。
* TODO: 一連の環境変数設定を整理する。useradd する？

```
$ SCP_NODE=192.168.70.33
$ KUBECONFIG=./kubeconfig
```

```
$ export PATH=$(pwd)/bin:$PATH
$ which kubectl
~/k8s-demo-vsphere-wcp/bin/kubectl
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"16+", GitVersion:"v1.16.7-2+bfe512e5ddaaaa", GitCommit:"bfe512e5ddaaaa7243d602d5d161fa09a57ecf3c", GitTreeState:"clean", BuildDate:"2020-03-03T03:40:35Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
$ kubectl-vsphere version
kubectl-vsphere: version 0.0.1, build 15839988, change 7857139
```

# 【WCP】WCP / vSphere Pod を使ってみる。

## 【WCP】Demo 0: Namespace の作成。

まず、vSphere Client で Namespace を作成しておく。

## 【WCP】kubectl でログイン。

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

context を変更

```
$ kubectl config use-context lab-ns-01
Switched to context "lab-ns-01".
```


## 【WCP】Demo 1: Pod の作成。

Pod の作成。

```
$ kubectl run -it --image=centos:7 --labels="app=jumpbox" --generator=run-pod/v1 --rm jbox
If you don't see a command prompt, try pressing enter.
[root@jbox /]#
[root@jbox /]# cat /etc/centos-release
CentOS Linux release 7.8.2003 (Core)
```

vSphere Pod 内から、NIC（podvnic）を見てみる。
* TODO: コンテナイメージを変更して、はじめから ip コマンドが入っている状態にする。

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

コンテナから exit。→ Pod は自動削除される。（--rm により）

```
$ exit
```

Service を作成。

* Type が LoadBalancer で、EXTERNAL-IP が払いだされていることを確認。

```
$ kubectl apply -f 101_demo-svc.yml
$ kubectl get svc -n lab-ns-01
NAME       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
demo-svc   LoadBalancer   10.96.0.60   192.168.70.34   80:30688/TCP   70s
$ kubectl get svc
NAME       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
demo-svc   LoadBalancer   10.96.0.60   192.168.70.34   80:30688/TCP   2m29s
```

 Deployment を作成。

 * Deployment の Ready が 2/2 になることを確認。

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

Web ブラウザから demo-svc Service にアクセスしてみる。

* Tera Term であれば、表示された URL ダブルクリックでブラウザが開く。
* Nginx の Welcome ページが表示されるはず。

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

環境の初期化。service と deployment を削除しておく。

```
$ kubectl delete -f 101_demo-svc.yml -f 102_demo-dep.yml
service "demo-svc" deleted
deployment.apps "demo-deploy" deleted
```

## 【WCP】Demo 2: Service LoadBalancer の動作テスト。

service と deployment を作成する。

```
$ kubectl apply -f 201_kuard-svc.yml
service/kuard-svc created
$ kubectl apply -f 202_kuard-dep.yml
deployment.apps/kuard created
```

Service の作成、kuard の Pod がすべて READY になったことを確認しておく。

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

kuard-svc の External-IP に Web browser からアクセス。 

* ブラウザ更新して、3つの Pod（コンテナ。Web サーバ）に振り分けられることを確認する。

```
$ kubectl get svc kuard-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.35/
```

環境の初期化。

```
$ kubectl delete -f 201_kuard-svc.yml,202_kuard-dep.yml
service "kuard-svc" deleted
deployment.apps "kuard" deleted
```

## 【WCP】Demo 3: NetworkPolicy を使用してみる。

WIP

* kubect apply -f 30N_xxx.yml で NetworkPolicy を作成。
* NSX-T の DFW ルールを確認する。

## 【WCP】Demo 4: Cloud Native Storage (PVC) を使用してみる.

StorageClass の確認。
* あらかじめ、vSphere Client で、Namespace での 仮想マシン ストレージ ポリシー 設定をしておく。

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

Service の作成。

* CNS には直接関係ないが、Pod でにアクセスしてみるために作成しておく。
* StatefulSet は何度か再作成するので、あえて Yaml を独立させた。

```
$ kubectl apply -f 401_cns-demo-svc.yml
service/cns-demo-svc created
$ kubectl get svc
NAME           TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
cns-demo-svc   LoadBalancer   10.96.0.30   192.168.70.34   80:32655/TCP   13s
$ kubectl get svc cns-demo-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.34/
```

StatefulSet（CNS なし）を作成。

* Deploymnt の時との違いを説明。

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

いったん、StatefulSet を削除する。
* CNS を使う StatefulSet を再作成するため。

```
$ kubectl delete -f 402_demo-ss.yml
statefulset.apps "nginx-ss" deleted
```

YAML ファイルの作成。

* StorageClass の名前が環境（仮想マシン ストレージ ポリシー名）によって変わるため。

```
$ sed "s/vsan-ftt-0/vm-storage-policy-nfs-vsk/" 403_demo-ss_with-cns.yml > demo-ss_with-cns.yml
$ diff 403_demo-ss_with-cns.yml demo-ss_with-cns.yml
31c31
<         volume.beta.kubernetes.io/storage-class: "vsan-ftt-0"
---
>         volume.beta.kubernetes.io/storage-class: "vm-storage-policy-nfs-vsk"
```

StatefulSet を作成。（CNS あり）

* Cloud Native Storage（CNS）は、PersistentVolumeClaim（PVC）で使用する。
* TODO: _ss → _sts にリソース名を修正する。（API の略称にあわせる）

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

PVC の確認。

```
$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
demo-pvc-nginx-ss-0   Bound    pvc-cb23dfef-028b-4fdb-9435-d1c20e283834   2Gi        RWO            vm-storage-policy-nfs-vsk   3m28s
demo-pvc-nginx-ss-1   Bound    pvc-22048084-554e-43e1-a23e-1ea4c511db5b   2Gi        RWO            vm-storage-policy-nfs-vsk   3m8s
demo-pvc-nginx-ss-2   Bound    pvc-2dcc144b-1608-4a39-8036-790d3ef7056d   2Gi        RWO            vm-storage-policy-nfs-vsk   113s
```

PV は、WCP だと権限がなくて見えなそう。
* ただし PV は PVC からわかる。
* vSphere Clint から「クラウド ネイティブ ストレージ」から確認しておく。

```
$ kubectl get pv
Error from server (Forbidden): persistentvolumes is forbidden: User "sso:Administrator@vsphere.local" cannot list resource "persistentvolumes" in API group "" at the cluster scope
```

環境の初期化。（StatefulSet）

```
$ kubectl delete -f demo-ss_with-cns.yml
statefulset.apps "nginx-ss" deleted
```

環境の初期化。（PVC）
* PVC は StatefulSet とは別に削除する。

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

環境の初期化。（Service）

```
$ kubectl delete svc cns-demo-svc
service "cns-demo-svc" deleted
```

# 【TKC】Tanzu Kubernetes Cluster の作成。

## あらためてデモ環境の確認。

Namespace の作成されている様子。
* これからどこに TKC を作成するか確認する。

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

用意されている VirtualMachineClasses の確認。

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

用意されている VirtualMachineImages の確認。
* これは vSphere のコンテンツ ライブラリにある。（事前にサブスクライブ設定＆DL）

```
$ kubectl get virtualmachineimages
NAME                                                        AGE
ob-15957779-photon-3-k8s-v1.16.8---vmware.1-tkg.3.60d2ffd   22m
```

Kuberenets Cluster の定義をした YAML を用意する。

* StorageClass の名前を環境にあわせて修正している。

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

TKC デプロイ先の Namespace を決める。（コピペむけ変数）

```
$ NS=lab-ns-02
```

Tanzu kubernetes cluster の作成。

```
$ kubectl -n $NS apply -f tkg-cluster-01.yml
tanzukubernetescluster.run.tanzu.vmware.com/tkg-cluster-01 created
$ kubectl get -n lab-ns-02 tanzukubernetesclusters.run.tanzu.vmware.com
NAME             CONTROL PLANE   WORKER   DISTRIBUTION                     AGE   PHASE
tkg-cluster-01   3               3        v1.16.8+vmware.1-tkg.3.60d2ffd   29m   running
```

# 【TKC】Tanzu Kubernetes Cluster を使用してみる。


## 【TKC】Demo-1 Tanzu k8s Cluster へのログイン。

TKC にログイン。
* ダウンロード済みの専用 kubectl を利用する。
* 指定する NS は、

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

current-context を確認。

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

##【TKC】Demo-0: Namespace の作成

Namespace を作成する。

```
$ kubectl create ns demo
namespace/demo created
```

Pod 作成の準備。

* Deployment, StatefulSet などから Pod を作成するためには PodSecurityPolicy の設定が必要。
* サービスアカウントむけに、RoleBindig を作成する。

```
$ kubectl -n demo apply -f 001_rolebinding_demo.yaml
rolebinding.rbac.authorization.k8s.io/rolebind-default-privileged-ns_demo_serviceaccounts created
```

## 【TKC】demo-1: Pod を作成してみる。 

service と deployment を作成。

```
$ kubectl -n demo apply -f 101_demo-svc.yml,102_demo-dep.yml
service/demo-svc created
deployment.apps/demo-deploy created
```

作成されたことを確認する。

```
$ kubectl -n demo get all
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

Web ブラウザから、Service にアクセスしてみる。
* Nginx Welcome ページが表示される。

```
$ kubectl -n demo get svc demo-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.36/
$ curl http://192.168.70.36/
```

初期化。

```
$ kubectl -n demo delete -f 101_demo-svc.yml,102_demo-dep.yml
service "demo-svc" deleted
deployment.apps "demo-deploy" deleted
```

## 【TKC】demo-2: LB の確認。

現在の Context の namespace を確認。

```
$ kubectl config current-context
tkg-cluster-01
$ kubectl config get-contexts tkg-cluster-01
CURRENT   NAME             CLUSTER         AUTHINFO                                        NAMESPACE
*         tkg-cluster-01   192.168.70.34   wcp:192.168.70.34:administrator@vsphere.local
```

ここからはデフォルトのnamespace を変更する。
* namespace の指定を省略する。

```
$ kubectl config set-context --current --namespace=demo
Context "tkg-cluster-01" modified.
$ kubectl config get-contexts tkg-cluster-01
CURRENT   NAME             CLUSTER         AUTHINFO                                        NAMESPACE
*         tkg-cluster-01   192.168.70.34   wcp:192.168.70.34:administrator@vsphere.local   demo
```

Service と Deployment を作成。

```
$ kubectl apply -f 201_kuard-svc.yml,202_kuard-dep.yml
service/kuard-svc created
deployment.apps/kuard created
$ kubectl -n demo get svc kuard-svc --no-headers | echo "http://$(awk '{print $4}')/"
http://192.168.70.36/
```

Pod 起動を確認。

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

Web ブラウザからアクセス確認できたら、初期化。

```
$ kubectl delete -f 201_kuard-svc.yml,202_kuard-dep.yml
service "kuard-svc" deleted
deployment.apps "kuard" deleted
```

## 【TKC】demo-3: NetworkPolicy。

WIP

* NSX DFW ではなく、calico によるルールになる。Pod の iptables -L あたりで確認。

## 【TKC】demo-4: CNS を使用してみる。

StorageClass の確認。
* これは WCP（vSphere Pod） と同じ。

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

service を作成。

```
$ kubectl apply -f 401_cns-demo-svc.yml
service/cns-demo-svc created
```

StatefulSet 作成。（PVCなし）

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

StatefulSet をいったん削除。

```
$ kubectl delete -f 402_demo-ss.yml
statefulset.apps "nginx-ss" deleted
```

CNS を使用する StatefulSet の YAML を作成。

```
$ sed "s/vsan-ftt-0/vm-storage-policy-nfs-vsk/" 403_demo-ss_with-cns.yml > demo-ss_with-cns_tanzu.yml
$ diff 403_demo-ss_with-cns.yml demo-ss_with-cns_tanzu.yml
31c31
<         volume.beta.kubernetes.io/storage-class: "vsan-ftt-0"
---
>         volume.beta.kubernetes.io/storage-class: "vm-storage-policy-nfs-vsk"
```

StatefulSet を再作成。（with PVC）

```
$ kubectl apply -f demo-ss_with-cns_tanzu.yml
statefulset.apps/nginx-ss created
```

コンテナの起動確認。

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

PVC の確認。

```
$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
demo-pvc-nginx-ss-0   Bound    pvc-9531a4ff-8230-4e36-96bc-c395ea64b91c   2Gi        RWO            vm-storage-policy-nfs-vsk   5m31s
demo-pvc-nginx-ss-1   Bound    pvc-25cd1ed4-c48b-4296-bd70-9bd05d92b388   2Gi        RWO            vm-storage-policy-nfs-vsk   4m41s
demo-pvc-nginx-ss-2   Bound    pvc-dcafc47b-1fdc-41cf-9b81-e76bce12f906   2Gi        RWO            vm-storage-policy-nfs-vsk   3m39s
```

PV の確認。

* WCP のように制限されてなく見られる。

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS                REASON   AGE
pvc-25cd1ed4-c48b-4296-bd70-9bd05d92b388   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-1   vm-storage-policy-nfs-vsk            4m27s
pvc-9531a4ff-8230-4e36-96bc-c395ea64b91c   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-0   vm-storage-policy-nfs-vsk            5m40s
pvc-dcafc47b-1fdc-41cf-9b81-e76bce12f906   2Gi        RWO            Delete           Bound    demo/demo-pvc-nginx-ss-2   vm-storage-policy-nfs-vsk            3m52s
```

これでデモ終わり。初期化。

```
$ kubectl delete -f demo-ss_with-cns_tanzu.yml
statefulset.apps "nginx-ss" deleted
$ kubectl delete pvc --all
persistentvolumeclaim "demo-pvc-nginx-ss-0" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-1" deleted
persistentvolumeclaim "demo-pvc-nginx-ss-2" deleted
```

EOF
