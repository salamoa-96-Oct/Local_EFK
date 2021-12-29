# Local EFK 설치 가이드 (kubernetes 환경)
#### 설치일 2021-10
# 구성 환경(Version)

![Untitled](https://user-images.githubusercontent.com/83336692/147642899-67d26079-fb89-41a1-934c-b7f07d522bfc.png)


- 구동환경
    
    → virtual_box(Oracle)
    
    Ubuntu : 20.04 TLS 버전 사용
    
    Nat : 10.100.0.0/24  DHCP 사용
    
    → Kubernetes Version
    
    ```yaml
    $ kubectl cluster-info
    	Kubernetes control plane is running at https://10.100.0.104:6443
    	CoreDNS is running at https://10.100.0.104:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    	
    	To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    
    $ kubectl version
    	Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:38:50Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    	Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:32:41Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    
    $ kubeadm version
    	kubeadm version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:37:34Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    ```
    

→ helm version : 3.7.1 version 사용

→ elasticsearch version : 7.15

→ fluentd version : 3.3

→ kibana version : 7.15

---

# helm

- 2가지 설치하는 방식

```yaml
$ curl -o https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

	% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
	                                 Dload  Upload   Total   Spent    Left  Speed
	100 11148  100 11148    0     0  11564      0 --:--:-- --:--:-- --:--:-- 11552
	
$ bash ./get-helm-3

	Downloading https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
	Verifying checksum... Done.
	Preparing to install helm into /usr/local/bin
	helm installed into /usr/local/bin/helm
	
$ helm version

	version.BuildInfo{Version:"v3.7.1", GitCommit:"1d11fcb5d3f3bf00dbe6fe31b8412839a96b3dc4", GitTreeState:"clean", GoVersion:"go1.16.9"}
```

2번째 방법

```yaml
$ snap install helm --classic

	helm3.7.0 from Snapcrafters installed

$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

$ chmod 700 get_helm.sh

$ ./get_helm.sh
	Helm v3.7.1 is already latest
```

```yaml
repo 등록
ingress-nginx	https://kubernetes.github.io/ingress-nginx
elastic      	https://helm.elastic.co                   
fluent       	https://fluent.github.io/helm-charts      
kokuwa       	https://kokuwaio.github.io/helm-charts
```
# Elastic Search 설치

- Elastic Search 7.15 version 설치

```yaml
$ helm repo add elastic https://helm.elastic.co
	"elastic" has been added to your repositories

$ helm repo list
	NAME   	URL                    
	elastic	https://helm.elastic.co

$ mkdir EFK       -> helm 값을 불러올 디렉토리 생성

$ cd EFK
$ helm pull elastic/elasticsearch
$ ll
$ elasticsearch-7.15.0.tgz
$ tar xvf elasticsearch-7.15.0.tgz
```

helm pull을 이용해서 helm Chart를 가져옵니다.

- PVC 없이 생성한 모습

```yaml
$ helm install elasticsearch .
	W1111 12:41:56.666069   87874 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
	W1111 12:41:56.691510   87874 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
	NAME: elasticsearch
	LAST DEPLOYED: Thu Nov 11 12:41:56 2021
	NAMESPACE: default
	STATUS: deployed
	REVISION: 1
	NOTES:
	1. Watch all cluster members come up.
	  $ kubectl get pods --namespace=default -l app=elasticsearch-master -w2. Test cluster health using Helm test.
	  $ helm --namespace=default test elasticsearch

$ kubectl get all
	NAME                         READY   STATUS    RESTARTS   AGE
	pod/elasticsearch-master-0   0/1     Pending   0          106s
	pod/elasticsearch-master-1   0/1     Pending   0          106s
	pod/elasticsearch-master-2   0/1     Pending   0          106s
	
	NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
	service/elasticsearch-master            ClusterIP   10.110.254.69   <none>        9200/TCP,9300/TCP   106s
	service/elasticsearch-master-headless   ClusterIP   None            <none>        9200/TCP,9300/TCP   106s
	service/kubernetes                      ClusterIP   10.96.0.1       <none>        443/TCP             34d
	
	NAME                                    READY   AGE
	statefulset.apps/elasticsearch-master   0/3     106s

$ kubectl describe pod elasticsearch-master-0
	Events:
	  Type     Reason            Age                  From               Message
	  ----     ------            ----                 ----               -------
	  Warning  FailedScheduling  49s (x3 over 2m13s)  default-scheduler  0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims.
# PVC가 바운딩되지 않아서 오류가 나는 모습 이를 해결하기위해서 nfs-provision으로 해결
```

- nfs 연동과정

```yaml
master-node에서 nfs-server 구성

$ apt-get install -y nfs-kernel-server

# 공유할 디렉토리 생성
$ mkdir /nfs/
$ chmod -R 777 /nfs

$ vi /etc/exports
	/nfs *(rw,sync,no_subtree_check) 
#/[경로] [허용 IP대역](rw,sync,no_subtree_check)
#       rw: read and write operations
#       sync: write any change to the disc before applying it
#       no_subtree_check: prevent subtree checking

$ exports -a
# 적용
$ systemctl restart nfs-kernel-server.service
$ systemctl status nfs-kernel-server
# 재등록하고 확인
● nfs-server.service - NFS server and services
   Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
   Active: active (exited) since Thu 2021-11-11 12:50:49 KST; 23s ago
   Process: 93198 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
   Process: 93199 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
   Main PID: 93199 (code=exited, status=0/SUCCESS)

11월 11 12:50:47 master.example.com systemd[1]: Starting NFS server and services...
11월 11 12:50:49 master.example.com systemd[1]: Finished NFS server and services.

--------------------------------------------------------------------------------------
worker-node[1-2]에서 nfs-common 구성

$ apt-get install -y nfs-common
# service 설치
$ mkdir /nfs
# 공유할 디렉토리 생성
$ mount 10.100.0.104:/nfs /nfs

$ cd /nfs

$ cat > test
hello nfs
ctr + d

$ ll
	합계 12
	drwxrwxrwx  2 root   root    4096 11월 11 12:54 ./
	drwxr-xr-x 21 root   root    4096 11월 11 12:54 ../
	-rw-r--r--  1 nobody nogroup   14 11월 11 12:55 test

root@node1:~# cd /nfs/
root@node1:/nfs# ll
	합계 12
	drwxrwxrwx  2 root   root    4096 11월 11 12:54 ./
	drwxr-xr-x 21 root   root    4096 11월 11 12:53 ../
	-rw-r--r--  1 nobody nogroup   14 11월 11 12:55 test
	

root@master:/nfs# ll
합계 12
drwxrwxrwx  2 root   root    4096 11월 11 12:54 ./
drwxr-xr-x 21 root   root    4096 11월 11 12:47 ../
-rw-r--r--  1 nobody nogroup   14 11월 11 12:55 test

정상적으로 nfs-server가 mount되어 있는 상태
master node , worker node 2개
방화벽 해제 ufw disable
```

- nfs-provisioning

```yaml
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
# helm repo 등록(nfs)

$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
	"nfs-subdir-external-provisioner" has been added to your repositories

$ helm pull https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/releases/download/nfs-subdir-external-provisioner-4.0.14/nfs-subdir-external-provisioner-4.0.14.tgz

$ ll
	합계 20
	drwxr-xr-x 5 root root 4096 11월 11 13:02 ./
	drwx------ 9 root root 4096 11월 11 13:02 ../
	drwxr-xr-x 4 root root 4096 11월 11 12:40 elasticsearch/
	drwxr-xr-x 4 root root 4096 11월 11 13:02 nfs-subdir-external-provisioner/
	drwxr-xr-x 2 root root 4096 11월 11 13:02 tgz.file/

$ vim values.yaml
 
10 nfs:
 11   server: 10.100.0.104
		  # nfs-server-ip 
 12   # path: /nfs-storage -> nfs 경로
			path: /nfs
 13   mountOptions:
 14   volumeName: nfs-subdir-external-provisioner-root
 15   # Reclaim policy for the main nfs volume
 16   reclaimPolicy: Retain

27   # defaultClass: false
		   defaultClass: true
29   # Set a StorageClass name
30   # Ignored if storageClass.create is false
31   # name: nfs-client
			 name: nfs

```

values.yaml 값을 알맞게 수정 후 설치

```yaml
$ helm install nfs .
	NAME: nfs
	LAST DEPLOYED: Thu Nov 11 13:12:09 2021
	NAMESPACE: default
	STATUS: deployed
	REVISION: 1
	TEST SUITE: None

$ kubectl get all
NAME                                                       READY   STATUS    RESTARTS   AGE
	pod/nfs-nfs-subdir-external-provisioner-587d7658c4-nghbv   1/1     Running   0          23s
	
	NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   34d
	
	NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/nfs-nfs-subdir-external-provisioner   1/1     1            1           23s
	
	NAME                                                             DESIRED   CURRENT   READY   AGE
	replicaset.apps/nfs-nfs-subdir-external-provisioner-587d7658c4   1         1         1       23s

$ cd /EFK/elasticsearch
$ vim values.yaml
# 수정 nfs pvc 경로 설정
93 volumeClaimTemplate:
94   storageClassName: "nfs"
95   accessModes: ["ReadWriteOnce"]
96   resources:
97     requests:
98       storage: 10Gi

74 resources:
75   requests:
76     cpu: "200m"
77     memory: "500mi"
78   limits:
79     cpu: "200m"
80     memory: "500mi"

## --set volumeClaimTemplate.storageClassName=nfs 명령어로 지정해주는 걸 values에서 해결
## nfs provisoning yaml에서 설정한 storageClassName: nfs로 동일하게 지정
# 수정후 elasticsearch 다시 설치

$ helm upgrade --install elasticsearch .
$ kubectl get all

	NAME                                                       READY   STATUS     RESTARTS   AGE
	pod/elasticsearch-master-0                                 0/1     Init:0/1   0          36s
	pod/elasticsearch-master-1                                 0/1     Init:0/1   0          36s
	pod/elasticsearch-master-2                                 0/1     Pending    0          36s
	pod/nfs-nfs-subdir-external-provisioner-587d7658c4-nghbv   1/1     Running    0          20m
	
	NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
	service/elasticsearch-master            ClusterIP   10.102.121.181   <none>        9200/TCP,9300/TCP   37s
	service/elasticsearch-master-headless   ClusterIP   None             <none>        9200/TCP,9300/TCP   37s
	service/kubernetes                      ClusterIP   10.96.0.1        <none>        443/TCP             34d
	
	NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/nfs-nfs-subdir-external-provisioner   1/1     1            1           20m
	
	NAME                                                             DESIRED   CURRENT   READY   AGE
	replicaset.apps/nfs-nfs-subdir-external-provisioner-587d7658c4   1         1         1       20m
	
	NAME                                    READY   AGE
	statefulset.apps/elasticsearch-master   0/3     37s

$ kubectl describe get pod elasticsearch-0
	Warning  FailedMount       66s   kubelet            MountVolume.SetUp failed for volume "kube-api-access-t6856" : write /var/lib/kubelet/pods/8e659fa3-2326-4fec-a099-20e74c4517e2/volumes/kubernetes.io~projected/kube-api-access-t6856/..2021_11_11_04_32_13.027730699/ca.crt: no space left on device
# 이 오류를 해결하기 위한 TS 방법
# /etc/kubernetes/manifests/kube-apiserver.yaml 수정
# NFS-provisioner 사용 및 default storage class 바인딩을 위해 수정

$ vi /etc/kubernetes/manifests/kube-apiserver.yaml

- --feature-gates=RemoveSelfLink=false
# 추가

```

[nfs-subdir-external-provisioner/README.md at master · kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/charts/nfs-subdir-external-provisioner/README.md)

기존에 있는 el 삭제후 다시 설치

```yaml
EFK_SK와는 다르게 values.yaml을 수정해서 설치합니다.

```yaml
158 # By default this will make sure two pods don't end up on the same node
159 # Changing this to a region would allow you to spread pods across regions
160 antiAffinityTopologyKey: "elasticsearch"
161 
162 # Hard means that by default pods will only be scheduled if there are enough nodes for them
163 # and that they will never end up on the same node. Setting this to soft will do this "best effort"
164 antiAffinity: "soft"
```
$ helm install elasticsearch .
```

```yaml
$ kubectl get all
	NAME                                                       READY   STATUS    RESTARTS   AGE
	pod/elasticsearch-master-0                                 1/1     Running   0          7m28s
	pod/elasticsearch-master-1                                 1/1     Running   0          7m28s
	pod/elasticsearch-master-2                                 1/1     Running   0          7m28s
	pod/nfs-nfs-subdir-external-provisioner-587d7658c4-k9428   1/1     Running   0          25m
	
	NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
	service/elasticsearch-master            ClusterIP   10.100.120.32   <none>        9200/TCP,9300/TCP   7m28s
	service/elasticsearch-master-headless   ClusterIP   None            <none>        9200/TCP,9300/TCP   7m28s
	service/kubernetes                      ClusterIP   10.96.0.1       <none>        443/TCP             34d
	
	NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/nfs-nfs-subdir-external-provisioner   1/1     1            1           25m
	
	NAME                                                             DESIRED   CURRENT   READY   AGE
	replicaset.apps/nfs-nfs-subdir-external-provisioner-587d7658c4   1         1         1       25m
	
	NAME                                    READY   AGE
	statefulset.apps/elasticsearch-master   3/3     7m28s
```

[helm-charts/README.md at main · elastic/helm-charts](https://github.com/elastic/helm-charts/blob/main/elasticsearch/README.md)

---

# Fluentd 설치

- repo 등록

```yaml
$ helm repo add kokuwa https://kokuwaio.github.io/helm-charts

$ helm pull https://github.com/kokuwaio/helm-charts/releases/download/fluentd-elasticsearch-13.1.0/fluentd-elasticsearch-13.1.0.tgz

$ vim values.yaml
	53   # hosts: ["elasticsearch-client:9200"]
	53   hosts: ["elasticsearch-master:9200"]

$ helm install fluentd .

$ kubectl get pod
	NAME                                                   READY   STATUS    RESTARTS   AGE
	elasticsearch-master-0                                 1/1     Running   0          26m
	elasticsearch-master-1                                 1/1     Running   0          26m
	elasticsearch-master-2                                 1/1     Running   0          26m
	fluentd-fluentd-elasticsearch-8v8mx                    1/1     Running   0          91s
	fluentd-fluentd-elasticsearch-rbq6s                    1/1     Running   0          91s
	nfs-nfs-subdir-external-provisioner-587d7658c4-k9428   1/1     Running   0          44m
```

---

# Kibana 설치

- repo 등록

```yaml
$ helm repo add kibana https://helm.elastic.co
$ helm install kibana/kibana

$ kubectl get pod
	
	NAME                                                   READY   STATUS    RESTARTS   AGE
	elasticsearch-master-0                                 1/1     Running   0          34m
	elasticsearch-master-1                                 1/1     Running   0          34m
	elasticsearch-master-2                                 1/1     Running   0          34m
	fluentd-fluentd-elasticsearch-8v8mx                    1/1     Running   0          10m
	fluentd-fluentd-elasticsearch-rbq6s                    1/1     Running   0          10m
	kibana-kibana-6b58f7b66c-lph5n                         1/1     Running   0          2m48s
	nfs-nfs-subdir-external-provisioner-587d7658c4-k9428   1/1     Running   0          53m
```

---

## Ingress-controller

## 2.4. Ingress Controller 설치

### 2.4.1. Namespace 생성

```yaml
$ kubectl create namespace ingress-nginx
```

### 2.4.2. 설치

```yaml
$ helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.replicaCount=2
```
