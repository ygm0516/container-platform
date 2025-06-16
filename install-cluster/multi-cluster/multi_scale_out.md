## CSP 쿠버네티스 서비스 Istio 멀티 클러스터 구성 가이드(NAVER-NHN)

<br>

## Table of Contents
1. [문서 개요](#1)    
   1.1. [목적](#1.1)  
   1.2. [범위](#1.2)  

2. [정보](#2)  
   2.1. [Prerequisite](#2.1)  
   2.2. [배포 목록](#2.2)  
   2.3. [자원 확인](#2.3)  

3. [scale out](#3)  
   3.1. [NHN cluster scale out](#3.1)  
   3.1.1 [정상 작동 확인](#3.1.1)  
   3.2. [NCP cluster node scale out](#3.2)  
   3.2.1 [node(1/2) scale out](#3.2.1)     
   3.2.2 [정상 작동 확인](#3.2.2)     
   3.2.3 [node(2/2) scale out](#3.2.3)   
   3.2.4 [정상 작동 확인](#3.2.4)     

4. [scale in](#4)  
   4.1. [NHN cluster scale in](#4.1)  
   4.1.1 [정상 작동 확인](#4.1.1)  
   4.2. [NCP cluster node scale in](#4.2)  
   4.2.1 [node(1/2) scale in](#4.2.1)     
   4.2.2 [정상 작동 확인](#4.2.2)     
   4.2.3 [node(2/2) scale in](#4.2.3)   
   4.2.4 [정상 작동 확인](#4.2.4)     



<br><br>
   
## <span id='1'> 1. 문서 개요
### <span id='1.1'> 1.1. 목적
본 문서(`대표 포털 멀티 클러스터 scale out 가이드`)에서는 현재 대표 포털이 배포되어진 멀티 클러스터와 동일한 상태의 클러스터를 생성하여 scale out하는 테스트 과정과 방법을 기술하였다.

<br>

### <span id='1.2'>1.2. 범위
`CSP 쿠버네티스 서비스` 클러스터를 대상으로 Istio를 이용하여 `멀티 클러스터`를 구성하였다.
본 문서에서는 `nhn cloud`와 `naver cloud` 환경에서 scale out 테스트 과정과 방법을 기술하였다.



## <span id='2'> 2. 테스트 환경 정보

### <span id='2.1'>2.1. Prerequisite
현재 대표 포털이 배포되어있는 멀티 클러스터와 비슷한 환경을 재현하기 위해 샘플앱 배포하여 자원 많이 쓰는 상태에서 진행한다.

- 현재 운영 멀티 클러스터 자원 상태
```bash
#ncp-cluster
ncloud@ncp-kpaas-nodepools-w-2hna:~$ kubectl config use-context ncp-kpaas-cluster
Switched to context "ncp-kpaas-cluster".
ncloud@ncp-kpaas-nodepools-w-2hna:~$ kubectl top node
NAME                         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
ncp-kpaas-nodepools-w-2hna   193m         4%     9019Mi          67%       
ncp-kpaas-nodepools-w-2hnb   352m         8%     9602Mi          72%  

#nhn-cluster
ncloud@ncp-kpaas-nodepools-w-2hna:~$ kubectl config use-context nhn-kpaas-cluster
Switched to context "nhn-kpaas-cluster".
ncloud@ncp-kpaas-nodepools-w-2hna:~$ kubectl top node
NAME                                      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
nhn-kpaas-cluster-default-worker-node-0   1173m        29%    10609Mi         66%     
```


### <span id='2.2'>2.2. 베포 목록

- 대표포털 동일하게 배포

`kps-api-business`<br>
`kps-api-opperate`<br>
`kps-api-common`<br>
`kps-api-support`<br>
`kps-api-edu`<br>
`kps-api-content`<br>
`kps-web-user`<br>
`kps-web-admin`<br>
`kps-zuul-gateway`<br>

```bash
#배포 확인 명령어
```

### <span id='2.2'>2.3. 자원 확인
- 대표포털 동일한 자원 상태인지를 확인

```bash
#자원 상태 확인 명령어
```
## <span id='3'> 3. scale out
### <span id='3.1'> 3.1. NHN cluster scale out
> nhn cluster node spec 변경 가능

```bash

```
### <span id='3.1.1'> 3.1.1 정상 작동 확인
### <span id='3.2'> 3.2. NCP cluster node scale out
 ![ncp spec 변경 관련 Faq](image.png)
 >naver cloud 환경에서 기존에 사용하던 노드(IP)변경없이 scale out은 불가능함을 확인 <br>
 >따라서 변경할 스펙으로 노드를 추가한다음 기존에 사용중인 노드를 삭제해도 멀티 클러스터가 정상적으로 작동하는지를 확인하는 절차를 진행한다.

### <span id='3.2.1'> 3.2.1 node(1/2) scale out
### <span id='3.2.2'> 3.2.2 정상 작동 확인
### <span id='3.2.3'> 3.2.3 node(2/2) scale out
### <span id='3.2.4'> 3.2.4 정상 작동 확인

## <span id='3'> 3. scale in
### <span id='3.1'> 3.1. NHN cluster scale in
### <span id='3.1.1'> 3.1.1 정상 작동 확인
### <span id='3.2'> 3.2. NCP cluster node scale in
### <span id='3.2.1'> 3.2.1 node(1/2) scale in
### <span id='3.2.2'> 3.2.2 정상 작동 확인
### <span id='3.2.3'> 3.2.3 node(2/2) scale in
### <span id='3.2.4'> 3.2.4 정상 작동 확인