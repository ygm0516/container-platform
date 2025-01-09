# container platform ingress 설정 가이드.md

## 목차
1. [문서 개요](#1)
    * [1.1. 목적](#1-1)
    * [1.2. 범위](#1-2)
2. [오류 내용](#2)
    * [2.1. 오류 내용](#2-1)
3. [ingress 설정](#3)
    * [3.1. cp cluster 설치시 변수 설정](#3-1)
    * [3.2. ingress 설정 방법](#3-2)
4. [결과 확인](#4)
    * [4.1. 결과 확인](#4-1)


# <div id='1'/> 1. 문서 개요

## <div id='1-1'/> 1.1. 목적
본 문서는 (K-PaaS 컨테이너 플랫폼 클러스터 설치 가이드) 는 기획자, 개발자, 운영자 지원 환경의 개방형 PaaS 플랫폼인 K-PaaS 컨테이너 플랫폼의 클러스터를 설치시 ingress 설정과 관련하여 서술하였다.

## <div id='1-2'/> 1.1. 범위
설치 범위는 K-PaaS 컨테이너 플랫폼 환경의 기반이 되는 클러스터 설치를 단일 클라우드 환경 기준으로 작성하였다.


# <div id='2'/> 2. 오류 내용

## <div id='2-1'/>2.1. 오류 내용

K-PaaS 컨테이너 플랫폼 클러스터 설치에 필요한 환경변수 정보를 입력한다.
입력하는 과정에서 ingress 관련 설정을 하지 않고 임의값 (10.20.10.161)으로 입력한 다음 설치를 진행하였더니 클러스터 설치까지는 문제가 없었지만 cp-portal 설치시 vault, repository url 및 정보들을 가져오지 못하는 에러가 발생함




# <div id='3'/> 3. ingress 설정

## <div id='3-1'/>3.1. cp cluster 설치시 변수 설정
```
$ vi cp-cluster-vars.sh
```

<br>

Control Plane

|환경변수|설명|비고|
|---|---|---|
|KUBE_CONTROL_HOSTS|Control Plane 노드의 갯수||
|ETCD_TYPE|ETCD 배포 방식<br>external : 별도의 노드에 ETCD 구성<br>stacked : Control Plane 노드에 ETCD 구성|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정|
|LOADBALANCER_DOMAIN|사전에 구성한 로드밸런서의 VIP 또는 Domain 정보|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정|
|ETCD1_NODE_HOSTNAME|ETCD 1번 노드의 호스트명|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`ETCD_TYPE`** 값이 external 일 경우 설정|
|ETCD1_NODE_PRIVATE_IP|ETCD 1번 노드의 Private IP|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`ETCD_TYPE`** 값이 external 일 경우 설정|
|ETCD{n}_NODE_HOSTNAME|ETCD n번 노드의 호스트명|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`ETCD_TYPE`** 값이 external 일 경우 설정<br>**`KUBE_CONTROL_HOSTS`** 값만큼 설정|
|ETCD{n}_NODE_PRIVATE_IP|ETCD n번 노드의 Private IP|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`ETCD_TYPE`** 값이 external 일 경우 설정<br>**`KUBE_CONTROL_HOSTS`** 값만큼 설정|
|MASTER1_NODE_HOSTNAME|Control Plane 1번 노드의 호스트명||
|MASTER1_NODE_PUBLIC_IP|Control Plane 1번 노드의 Public IP|Control Plane 1번 노드만 Public IP 정보 필요|
|MASTER1_NODE_PRIVATE_IP|Control Plane 1번 노드의 Private IP||
|MASTER{n}_NODE_HOSTNAME|Control Plane n번 노드의 호스트명|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`KUBE_CONTROL_HOSTS`** 값만큼 설정|
|MASTER{n}_NODE_PRIVATE_IP|Control Plane n번 노드의 Private IP|**`KUBE_CONTROL_HOSTS`** 값이 2 이상일 경우 설정<br>**`KUBE_CONTROL_HOSTS`** 값만큼 설정|

<br>

Worker

|환경변수|설명|비고|
|---|---|---|
|KUBE_WORKER_HOSTS|Worker 노드의 갯수||
|WORKER1_NODE_HOSTNAME|Worker 1번 노드의 호스트명||
|WORKER1_NODE_PRIVATE_IP|Worker 1번 노드의 Private IP||
|WORKER{n}_NODE_HOSTNAME|Worker n번 노드의 호스트명|**`KUBE_WORKER_HOSTS`** 값만큼 설정|
|WORKER{n}_NODE_PRIVATE_IP|Worker n번 노드의 Private IP|**`KUBE_WORKER_HOSTS`** 값만큼 설정|

<br>

Storage

|환경변수|설명|비고|
|---|---|---|
|STORAGE_TYPE|Storage 정보<br>nfs : NFS 스토리지<br>rook-ceph : Rook Ceph 스토리지||
|NFS_SERVER_PRIVATE_IP|NFS Server 인스턴스의 Private IP|**`STORAGE_TYPE`** 값 nfs 일 경우 설정|

<br>

LoadBalancer Service

|환경변수|설명|비고|
|---|---|---|
|METALLB_IP_RANGE|MetalLB에서 사용할 Private IP 대역|Control Plane 노드와 동일한 네트워크 서브넷 대역 설정|
|INGRESS_NGINX_IP|MetalLB를 통해 Ingress Nginx Controller Service에서 사용할 ***`Private IP (인터페이스 일 경우) 또는 Public IP (로드밸런서 서비스 일 경우)`***|**`METALLB_IP_RANGE`** 값과 중복되지 않도록 설정|

<br>


처음에는 강제로 vault 및 몇몇 정보들을 강제로 가져올 수 있도록 스크립트 자체를 수정 -> 잘못된 방법

```
$ cd
$ vi cp-deployment/single/cp-cluster-vars.sh
```
>위의 파일에서 아래와 같이 정의함
>해당 부분을 cp-cluster 설치할 때 임의의 값으로 지정해서 설치함

`export INGRESS_NGINX_PRIVATE_IP=10.20.10.161`

>10.20.10.161로 ingress interface 생성 진행


## <div id='3-2'/>3.2. ingress 설정 방법

1. ** 스마일서브에서 한다고 가정 ( IaaS마다 방법이 달라질 수 있음 )
프로젝트 > 네트워크 > 네트워크 > 개요 > [내가 사용하는 네트워크] > 포트 생성 > 10.20.10.161 값으로 생성


2. 프로젝트 > 네트워크 > Floating IP> 프로젝트에 IP 할당 
생성한 포트 값(10.20.10.161)에 플로팅 ip 연결

3. 프로젝트 > compute > 인스턴스 

내가 생성한 cp-cluster에 만들어진 ingress IP값을 추가해주어야함
<아니면 control plain node에 생성한 생성한 인터페이스를 연결>
내가 생성한 클러스터가 yang-cluster1~5까지 일때 yang-cluster1> 인터페이스> 항목클릭> 허용된 주소쌍 추가> ingressip추가
nfs를 제외한 모든 cluster에 진행!

# <div id='4'/> 4. 결과 확인

## <div id='4-1'/>4.1. 결과 확인

```
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.35.181   10.20.10.161   80:31934/TCP,443:32388/TCP   12d
ingress-nginx-controller-admission   ClusterIP      10.233.46.27    <none>         443/TCP                      12d
```

##### 외부에서 EXTERNAL-IP curl 통신, 연결 불가
>floatingIP 값으로 연결해야됨

> floatingip를 붙이지 않아도 curl연결은 되는데 왜 굳이 floatingip를 붙여야하는 이유는? 
  curl 연결은 가능할수도 있지만 인터넷 url 접속은 안됨(외부망) 

##### 연결된 플로팅 ip로 외부에서 curl 통신, 404 에러 반환 시 정상 
```
$ curl http://[public ip]
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
