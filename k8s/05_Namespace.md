# Namespace

리소스를 분리
- 서비스 별
- 사용자 별
- 환경 : 개발, 스테이징, 프로덕션

> 서비스 : DNS 이름이 분리되는 용도
> RBAC : 권한을 NS에 설정

> https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/

``` bash
kubectl get namespaces
```

- kube-system: Kubernetes의 핵심 컴포넌트
- kube-public: 모든 사용자가 읽기 권한
- kube-node-lease: 노드의 HeartBeat 체크를 위한 Lease 리소스가 존재
- default: 기본 작업 공간

``` bash
kubectl create ns developments
```

``` bash
kubectl delete ns developments
```

``` bash
kubectl get pods -A | --all-namespaces
```

``` bash
kubectl get pods -n kube-system
```


`ns-dev.yaml`
``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

``` bash
kubectl create -f ns-dev.yaml
```

`myweb-dev.yaml`
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  namespace: dev
spec:
  containers:
    - name: myweb
      image: httpd
      ports:
        - containerPort: 80
          protocol: TCP          
```

``` bash
kubectl create -f myweb-dev.yaml
```

``` bash
kubectl delete -f myweb-dev.yaml
```