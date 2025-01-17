# Volume

`spec.volumes.*` : 볼륨 유형

## emptyDir

임시로 사용할 빈 볼륨, 파드 삭제 시 볼륨 같이 삭제

Pod가 실행되는 노드의 디스크 공간에 마운트 된다. Pod의 컨테이너 간에 볼륨을 공유하기 위해 사용된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb-pod
spec:
  containers:
    - name: myweb1
      image: httpd
      volumeMounts:
        - name: emptyvol
          mountPath: /empty
    - name: myweb2
      image: ghcr.io/c1t1d0s7/go-myweb:alpine
      volumeMounts:
        - name: emptyvol
          mountPath: /empty
  volumes:
    - name: emptyvol
      emptyDir: {}
```

``` bash
kubectl exec -it myweb-pod -c myweb1 -- bash

> cd /empty
> touch a b c
```

``` bash
kubectl exec -it myweb-pod -c myweb2 -- sh

> ls /empty
```

emptyDir은 한 컨테이너가 파일을 관리하고 한 컨테이너가 그 파일을 사용하는 경우에 유용하게 사용할 수 있다. GitHub 코드를 받아와 애플리케이션 컨테이너에 공유해주는 Side-Car 컨테이너가 될 수 있고, 설정 파일을 동적으로 갱신하는 컨테이너를 Pod에 포함시킬 수도 있다.

## initContainer(초기화 컨테이너)

> https://kubernetes.io/ko/docs/concepts/workloads/pods/init-containers/

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
    - name: gitpull
      image: alpine/git
      args:
        - clone
        - -b
        - v2.18.1
        - https://github.com/kubernetes-sigs/kubespray.git
        - /repo
      volumeMounts:
        - name: gitrepo
          mountPath: /repo
  containers:
    - name: gituse
      image: busybox
      args:
        - tail
        - -f
        - /dev/null
      volumeMounts:
        - name: gitrepo
          mountPath: /kube
  volumes:
    - name: gitrepo
      emptyDir: {}
```

## hostPath

hostPath는 노드의 파일시스템의 특정 파일이나 디렉토리를 Pod에 마운트하여 사용한다. 호스트와 볼륨을 공유하기 위해 사용된다.

`/mnt/web_contents/index.html`

``` html
<h1> Hello hostPath </h1>
```

> 참고
> 로컬 스토리지 : 다른 호스트에 스토리지 볼륨을 제공할 수 없다.
> - emptyDir
> - hostPath
> - gitRepo
> - local

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
  labels:
    name: hostpath-pod
spec:
  containers:
  - name: hostpath-pod
    image: busypox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: my-hostpath-volume
      mountPath: /etc/data
  volumes:
  - name: my-hostpath-volume
    hostPath:
      path: /tmp
```

Pod를 생성한 뒤 Pod의 컨테이너 내부로 들어가 `/etc/data` 디렉토리에 파일을 생성하면 호스트의 `/tmp` 디렉토리에 파일이 저장된다.
하지만, Deployment의 Pod에 장애가 생겨 다른 노드로 Pod가 옮겨갔을 경우, 이전 노드에 저장된 데이터를 사용할 수 없다.


## PV & PVC

Pod 내부에서 특정 데이터를 보유해야 하는 statefule한 app의 경우 stateless한 데이터를 영속적으로 저장하기 위한 방법이 필요하다.

Pod에서 실행중인 애플리케이션이 디스크에 데이터를 유지해야한다면 emptyDir, hostPath를 사용할 수 없다.

어떤 클러스터 노드에서도 접근할 수 있어야 하므로 NAS 유형의 스토리지에 저장이 되어야 한다.


- PersistentVolyme : 스토리지 볼륨 정의

PV는 Volume 자체를 의미한다.

Kubernetes Cluster에서 관리되는 저장소로 Pod와는 별개로 관리되고 별도의 생명주기를 가지고 있어 Pod가 재실행되더라도 PV 데이터는 정책에 따라 유지/삭제된다.

특징
  - Cluster Storage의 일부
  - Cluster Node와 같은 리소스
  - Namespace에 속하지 않음
  - Pod와 독립적인 lifeCycle을 가짐


- PersistentVolumeClaim : PV를 요청

PVC는 Pod의 볼륨과 PVC를 연결하는 관계 선언이다.

PVC는 사용자가 PV에 요청으로 PV를 추상화해 개발자가 손쉽게 PV를 사용할 수 있도록 해주는 기능이다.

사용하고 싶은 용량은 얼마인지 읽기/쓰기는 어떤 모드로 설정하고 싶은지 등을 정해서 PV에게 전달하는 역할을 한다.

개발자는 Pod를 생성할 때 Volume을 정의하고 이 볼륨 정의 부분에 물리적 디스크에 대한 특성을 정의하는 것이 아닌 PVC를 지정해 관리자가 생성한 PV와 연결한다.

Storage를 Pod에 직접 할당하는 것이 아닌 중간에 PVC를 통해 사용하기 때문에 Pod와 Storage 관리를 명확히 구분할 수 있다.

예를 들어 가동 중인 Pod의 Storage를 변경하기 위해 Pod 자체를 재시작, 재생성 할 필요 없이 Pod에 연결된 PVC만 수정하면 Pod와 별개로 PVC를 통해 Storage를 관리할 수 있다.

특징
  - Storage에 대한 사용자의 요청
  - PV Resource 소비
  - 특정 Size나 Access mode를 요청


PV, PVC 예제

Pod
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: mypod
      image: httpd
      volumeMounts:
        - name: myvol
          mountPath: /tmp
  volumes:
    - name: myvol
      persistentVolumeClaim:
        name: mypvc
```

PVC
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  volumeName: mypv
```

PV
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  hostPath:
    path: /web_contents
    type: DirectoryOrCreate
```

### PV, PVC 생명주기

PV <- 1:1 -> PVC

1. 프로비저닝
  - 저장할 수 있는 공간 확보
    - Static Provisioning
    - Dynamic Provisioning
2. 바인딩
  - 상위의 Provisioning을 통해 만들어진 PV를 PVC와 연결하는 단계
3. 사용
4. 회수/반환(Reclaim)
  - Retain : 보존 - PV를 삭제하지 않음(Release <- PVC가 연결 X)
  - Delete : 삭제 - PV를 삭제 / 실제 스토리지 내용 삭제
  - Recycle : 재사용 (X) - 실제 스토리지 내용을 비우고, PV를 사용가능한 상태(Available)

### 접근 모드(Access Mode)
- ReadWriteOnce : RWO
- ReadWriteMany : RWX
- ReadOnlyMany : ROW

### Volume 모드

- filesystem : default 옵션으로 volume을 일반 파일시스템 형식으로 붙여서 사용하게 한다.
- raw : volume을 RAW 파일시스템 형식으로 붙여서 사용하게 한다.
- block : Filesystem이 없는 Block 장치와 연결될 때는 Block으로 설정한다.


### NFS를 사용한 정적 프로비저닝(Static Provision)

node1 : NFS 서버

``` bash
sudo apt install nfs-kernel-server -y
```

``` bash
sudo mkdir /nfsvolume
echo "<h1> Hello NFS Volume </h1>" | sudo tee /nfsvolume/index.html
```

``` bash
sudo chown -R www-data:www-data /nfsvolume
```

NFS의 경우 노드가 3개 있고 nfs volume을 공유할 때 파드를 생성하고 파드에 직접적으로 연결하는 것이 아니라 호스트에 마운팅으 시켜서 호스트의 kubelet이 제공하는 형태이다.

따라서 허용하는 대역을 `192.168.100.0/24` 호스트의 네트워크 대역으로 지정해야 한다.

`/etc/exports`

```
/nfsvolume 192.168.100.0/24(rw,sync,no_subtree_check,no_root_squash)
```
``` bash
sudo systemctl restart nfs-kernel-server
systemctl status nfs-kernel-server
```

node1, node2, node3

``` bash
sudo apt install nfs-common -y
```

또는

``` bash
ansible all -i ~/kubespray/inventory/mycluster/inventory.ini -m apt -a 'name=nfs-common' -b
```

PV
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1G
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfsvolume
    server: 192.168.100.100
```


PVC
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1G
  storageClassName: ''
  volumeName: mypv
```

RS
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myweb-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: myweb
          image: httpd
          volumeMounts:
            - name: myvol
              mountPath: /usr/local/apache2/htdocs
      volumes:
        - name: myvol
          persistentVolumeClaim:
            claimName: mypvc
```

SVC
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb-svc-lb
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: web
```

### EBS를 PV으로 사용


``` bash
export VOLUME_ID=$(aws ec2 create-volume --size 5 \
--region ap-northeast-2 \
--availability-zone ap-northeast-2a \
--volume-type gp2 \
--tag-specifications \
'ResourceType=volume, Tags=[{Key=KubernetesCluster, Value=mycluster.k8s.local}]' \
| jq '.VolumeId' -r)
```

`PV`
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 5Gi         # 이 볼륨의 크기는 5G
  accessModes:
    - ReadWriteOnce    # 하나의 포드 (또는 인스턴스) 에 의해서만 마운트
  awsElasticBlockStore:
    fsType: ext4
    volumeID: <VOLUME_ID>
  persistentVolumeReclaimPolicy: Delete # 연결된 PVC를 삭제함으로써 PV 삭제
```

`PVC, Pod`
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc                  # 1. my-ebs-pvc라는 이름의 pvc 를 생성
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce       # 2.1 속성이 ReadWriteOnce인 퍼시스턴트 볼륨과 연결
  resources:
    requests:
      storage: 5Gi          # 2.2 볼륨 크기가 최소 5Gi인 퍼시스턴트 볼륨과 연결
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-mount-container
spec:
  containers:
    - name: ebs-mount-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: ebs-volume
        mountPath: /mnt
  volumes:
  - name : ebs-volume
    persistentVolumeClaim:
      claimName: my-ebs-pvc    # 3. my-ebs-pvc라는 이름의 pvc를 사용
```

이전에 생성한 NFS는 1:N 마운트가 가능하지만, AWS의 EBS는 1:1 마운트만 가능하다. 또한 EBS는 생성 당시에 설정했던 크기만큼만 데이터를 저장할 수 있다. 

## 동적 프로비저닝

Pod, PVC, PV를 모두 지운 상태에서 NFS 스토리지를 다시 사용해야 한다면 매번 PV, PVC를 만들고 Pod를 연결시켜야하는 작업을 반복해야 한다.

동적 프로비저닝을 사용하면 PVC를 만들면 PVC에 바운딩 될 PV를 자동으로 만들어준다. 그리고 PV가 실제 스토리지에 연결된다.

StorageClass는 PV를 만들기 위한 템플릿을 가지고 있다. PVC를 만들 때 StorageClass를 지정하면 해당 StorageClass의 정보를 가지고 PV를 만들어서 스토리지에 연결시킨다. 관리자는 StorageClass를 만들어 놓으면 되고, 사용자는 PVC를 만들 때 어떤 PVC를 사용할 것인지를 결정만 하면 된다. 그러면 해당 StorageClass에서 PV를 자동으로 만들어준다.

### StorageClass

PVC를 정의하면 PVC의 내용에 따라서 Kubernetes 클러스터가 물리 Disk를 생성하고, 이에 연결된 PV를 생성한다.

디스크를 생성할 때 필요한 디스크의 타입을 정의할 수 있는데, 이를 StorageClass라고 하고, PVC에서 StorageClass를 지정하면 이에 맞는 디스크를 생성하도록 한다.

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

각 StorageClass에는 해당 StorageClass에 속하는 PV를 동적으로 프로비저닝할 때 사용되는 provisioner, parameters와 reclaimPolicy 필드가 포함된다.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-sc
spec:
  storageClassName: standard # storageClassName 항목을 정의하지 않았을 떄, k8s에서 기본 sc가 있다면 기본 sc로 수행
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```


### NFS Dynamic Provisioner 구성
> https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

``` bash
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
```

``` bash
cd nfs-subdir-external-provisioner/deploy
```

``` bash
kubectl create -f rbac.yaml
```

`deployment.yaml`
``` yaml
...
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.100.100
            - name: NFS_PATH
              value: /nfsvolume
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.100.100
            path: /nfsvolume
```

``` bash
kubectl create -f deployment.yaml
```

``` bash
kubectl create -f class.yaml
```

`mypvc-dynamic.yaml`
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1G
  storageClassName: 'nfs-client'
```

``` bash
kubectl create -f mypvc-dynamic.yaml
```

``` bash
echo "<h1> Hello NFS Dynamic Provision </h1>" | sudo tee /nfsvolume/XXX/index.html
```

`myweb-rs-dynamic.yaml`
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myweb-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: myweb
          image: httpd
          volumeMounts:
            - name: myvol
              mountPath: /usr/local/apache2/htdocs
      volumes:
        - name: myvol
          persistentVolumeClaim:
            claimName: mypvc-dynamic
```

``` bash
kubectl create -f myweb-rs-dynamic.yaml
```

### 기본 스토리지 클래스
`~/nfs-subdir-external-provisioner/deploy/class.yaml`
``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

상기의 `annotation`으로 기본 스토리지 클래스로 사용할 수 있다.

``` bash
kubectl apply -f class.yaml
```

```
kubectl get sc

NAME                   ...
nfs-client (default)   ...
```

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1G
```

## PV & PVC, NFS를 이용한 Jenkins CI Pod 구축

`jenkins.yaml`

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-admin
      securityContext:
            fsGroup: 1000 
            runAsUser: 1000
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "500Mi"
              cpu: "500m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins
              mountPath: /var/jenkins_home         
      volumes:
        - name: jenkins
          persistentVolumeClaim:
            claimName: jenkins

# Service Config
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '8080'
spec:
  selector: 
    app: jenkins
  type: NodePort  
  ports:
    - name: httpport
      port: 8080
      targetPort: 8080
      nodePort: 32000
    - name: jnlpport
      port: 50000
      targetPort: 50000
```

`pv.yaml`

``` yaml
# Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins
spec:
  capacity:
    storage: 15G
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /home/vagrant/nfs/jenkins
    server: 192.168.100.100
```

`pvc.yaml`

``` yaml
# Persistent Volume Claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: jenkins
  storageClassName: ''
```

`service-account.yaml`
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins
  namespace: default
  labels:
    "app.kubernetes.io/name": 'jenkins'
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-role-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: default
```

> **`jenkins-deployment`에 사용된 Security Context**
> 
> Security Context(보안 컨텍스트)는 K8s의 Pod나 컨테이너에 대한 접근 제어 설정이나 특수 권한을 설정하는 기능을 제공한다. 컨테이너 내에서 동작하는 프로세스의 사용자 ID나 그룹 ID를 설정하거나, 프로세스에 커널에 대한 접근 권한을 부여하는 것과 같은 기능을 할 수 있다.
>
> 상기 예제는 UID 1000, GID 1000으로 지정되어 있다. 따라서 `root` 사용자가 아닌 `jenkins` 사용자로 구동이 된다.
> 
> jenkins pod에 접속하여 `ps -ef` 명령을 확인해보면, UID가 jenkins로 생성되어 있는 것을 확인할 수 있다.
> 
> 이점: 보안 강화, 격리, 호환성 -> 시스템에 액세스하여 모든 작업을 수행하거나 다른 컨테이너의 침해 방지, 리소스 제한 우회 방지 등

## 호스트 노드의 파일 시스템 용량 초과

특정 Pod의 로그가 계속 쌓여 Pod가 할당된 노드의 디스크 용량이 부족해지는 현상이 발생하였다.

Kubernetes는 자원이 부족한 노드가 발생하면 자동으로 해당 노드의 Pod를 자원의 여유가 있는 다른 노드로 이전한다.

이 경우 애플리케이션 에러가 아닌, 노드의 문제이므로 `kubectl get events`로 확인 가능하며, 디버깅을 통해 문제 해결이 가능하다.