## CSP 쿠버네티스 서비스 Istio 멀티 클러스터 구성 가이드(NAVER-NHN)

<br>

## Table of Contents
1. [문서 개요](#1)    
   1.1. [목적](#1.1)  
   1.2. [범위](#1.2)  
   1.3. [시스템 구성도](#1.3)  
   1.4. [참고 자료](#1.4)
2. [정보](#2)  
   2.1. [Prerequisite](#2.1)  
   2.2. [설치 목록](#2.2)  
   2.3. [방화벽 정보](#2.3)  
   2.4. [시연 클러스터 환경](#2.4)  
3. [컨테이너 플랫폼 포털 배포 시 사전 설정](#3)    
   3.1. [Deployment 파일 다운로드](#3.1)   
   3.2. [도구 설치](#3.2)  
4. [Istio 멀티 클러스터 구성](#4)  
   4.1. [멀티 클러스터 접근 구성](#4.1)   
   4.2. [StorageClass 설정(참고)](#4.2)  
   4.3. [Metrics Server 설치(참고)](#4.3)  
   4.4. [Cilium CNI 사용 클러스터 설정 변경(참고)](#4.4)    
   4.5. [Istio 멀티 클러스터 구성 스크립트 실행](#4.5)   
5. [샘플 애플리케이션 배포](#5)  
   5.1. [멀티 클러스터 샘플 애플리케이션 배포](#5.1)     
   5.2. [멀티 클러스터 통신 테스트](#5.2)    
6. [Istio 멀티 클러스터 구성 삭제](#6)  

<br>

## <span id='1'> 1. 문서 개요
### <span id='1.1'> 1.1. 목적
본 문서(`CSP 쿠버네티스 서비스 Istio 멀티 클러스터 구성 가이드`)는 클라우드 서비스 제공업체의 Kubernetes 서비스에 Istio 멀티 클러스터 구성 방법을 기술하여 독립적인 Kubernetes 클러스터의 서비스 간 통신이 가능하도록 하는 방법을 기술한다. 

<br>

### <span id='1.2'>1.2. 범위
`CSP 쿠버네티스 서비스` 클러스터를 대상으로 Istio를 이용하여 `멀티 클러스터`를 구성하도록 작성하였다.<br>
본 문서에서는 `nhn cloud`와 `naver cloud` 환경을 사용하여 구성하는 방법을 작성하였다.

<br>

### <span id='1.3'>1.3. 시스템 구성도
시스템 구성은 서로 다른 네트워크인 N개의 Kubernetes 클러스터에 Istio Control Plane을 설치하여 Istio Mesh를 구성한다.<br>
클러스터 cluster N의 기본 네트워크는 network N으로 설정되며 클러스터 Pod 간의 직접적인 통신 없이 Istio Gateway를 통해 클러스터는 통신한다.

> *이미지 출처* <br>
> [Install Multi-Primary on different networks](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)<br>

<img src="https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/arch.svg" width="800">


<br>

### <span id='1.4'>1.4. 참고 자료
> [Istio Install Multi-Primary on different networks](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)<br>

<br>

## <span id='2'> 2. 정보
### <span id='2.1'>2.1. Prerequisite
- 작업 인스턴스는 **Ubuntu 22.04** 환경에서 진행하는 것을 기준으로 한다.
- 각 클러스터는 **로드 밸런서(LoadBalancer)** 유형의 서비스 지원이 필요하다.

<br>

### <span id='2.2'>2.2. 설치 목록
설치되는 도구 목록은 아래와 같다.
| 도구 | 버전 |
| :---: | :---: |  
| kubectl | v1.30.3 |
| Helm | v3.14.2 |
| step | 0.24.4 |
| Podman | - |
| ca-certificates | - |
| Istio | 1.24.0 |

| Istio 버전 | [Kubernetes 지원 버전](https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases)|  
| :---: | :---: |  
| `1.24` |1.28, 1.29, 1.30, 1.31|  

<br>

### <span id='2.3'>2.3. 방화벽 정보
IaaS Security Group의 열어줘야할 Port를 설정한다.
|프로토콜|포트|비고|
| :---: | :---: | --- | 
|TCP|15021|Istio ingressgateway status|
|TCP|80|Istio ingressgateway http|
|TCP|443|Istio ingressgateway https|
|TCP|31400|Istio ingressgateway tcp|
|TCP|15443|Istio ingressgateway mtls|
|TCP|15012|Istio ingressgateway istiod|
|TCP|15017|Istio ingressgateway webhook|


<br>


### <span id='2.4'>2.4. 시연 클러스터 환경
Istio를 활용하여 **`2개의 클러스터`** 를 기반으로 멀티 클러스터 환경을 구성하는 예제를 제공한다.

#### CSP 쿠버네티스 클러스터 환경
| Kubernetes Service | Version |CNI| Server Image |
| --- | --- | --- | --- |
| Ncloud Kubernetes Service (NKS) | v1.32.3 |Cilium| Ubuntu 24.04 |
| NHN kubernetes Service (NKS) | v1.32.3 |Calico| Ubuntu 22.04 |

<br>

## <span id='3'>3. 컨테이너 플랫폼 포털 배포 시 사전 설정
>csp에 따른 kubernetes service 생성 방법은 생략한다.(`csp에 따른 가이드 참고`) 

해당 가이드를 통해 구성된 Istio 멀티 클러스터 환경에 컨테이너 플랫폼 포털을 배포할 경우 아래 사전 설정이 필요하다.
<br>
>ncloud 확인 사항<br>
 - 메인 계정으로 인스턴스 생성 시 root, 서브 계정으로 인스턴스 생성 시 ncloud 계정을 기본으로 사용한다. 
    - 멀티 구성의 경우 각 vm에 설치하는 과정에서 동일한 계정이 존재하지 않으면 정상적으로 설치가 진행되지 않는다.
      - vm에 미리 동일한 계정을 생성해준다.
      - ex ) ubuntu -> ubuntu / ncloud -> ncloud
    -  sudo 실행 권한 확인
  
```bash
#계정 생성
$ sudo useradd -m -s /bin/bash ubuntu
$ echo "ubuntu ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers

$ sudo mkdir -p /home/ubuntu/.ssh
$ sudo ssh-keygen -t rsa -m PEM -N '' -f /home/ubuntu/.ssh/id_rsa
$ sudo cat /home/ubuntu/.ssh/id_rsa.pub | sudo tee -a /home/ubuntu/.ssh/authorized_keys
$ sudo chown -R ubuntu:ubuntu /home/ubuntu/.ssh

#비밀번호 변경
$ sudo passwd ubuntu
New password: 
Retype new password: 
passwd: password updated successfully
```
ubuntu로 계정 생성을 완료한 다음 ubuntu계정으로 다시 접근을 하여 다음 작업들을 진행한다.

### <span id='3.1'>3.1. Deployment 파일 다운로드
Istio 멀티 클러스터 구성을 위해 컨테이너 플랫폼 포털 Deployment 파일을 다운로드 받아 아래 경로로 위치시킨다.<br>

+ 컨테이너 플랫폼 포털 Deployment 파일 다운로드 :
  [cp-portal-deployment-v1.6.1.tar.gz](https://nextcloud.k-paas.org/index.php/s/FQFddRC4wiq5cdj/download)

```bash
# Deployment 파일 다운로드 경로 생성
$ mkdir -p ~/workspace/container-platform
$ cd ~/workspace/container-platform

# Deployment 파일 다운로드 및 파일 경로 확인
$ wget --content-disposition https://nextcloud.k-paas.org/index.php/s/FQFddRC4wiq5cdj/download

$ ls ~/workspace/container-platform
  cp-portal-deployment-v1.6.1.tar.gz

# Deployment 파일 압축 해제
$ tar -xvf cp-portal-deployment-v1.6.1.tar.gz
```


### <span id='3.2'>3.2. 도구 설치
> [[2.1. 설치 목록]](#2.1) <br>

Istio 멀티 클러스터 구성을 위해 필요한 커맨드 라인 등 도구의 사전 설치가 필요하다. <br>
`kubectl`의 경우 클러스터의 마이너(minor) 버전 차이 내에 있는 버전 사용이 필요하므로 클러스터 버전을 확인하여<br>
변경이 필요한 경우 아래 파일을 수정한다.<br>
또한 도구 설치는 `모든 클러스터`에서 사전에 진행한다.
```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ vi istio-vars-mc.sh
```
```bash
# 설치할 버전으로 변경 
# command line tool
KUBECTL_VERSION="1.30.3"
HELM_VERSION="3.14.2"
STEP_VERSION="0.24.4"

# Istio
ISTIO_VERSION="1.24.0"
...
```
도구 설치 스크립트를 실행한다.
```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ chmod +x install_tools.sh
$ ./install_tools.sh
```

<br>




## <span id='4'>4. Istio 멀티 클러스터 구성
Istio를 활용하여 **`2개의 클러스터`** 를 기반으로 멀티 클러스터 환경을 구성하는 예제를 제공한다.

### <span id='4.1'>4.1. 멀티 클러스터 접근 구성
Istio 멀티 클러스터 구성은 `kubectl config get-contexts` 명령어로 출력된 컨텍스트 목록을 기반으로 진행된다.<br>
따라서 각 클러스터에 접근할 수 있도록 컨텍스트를 미리 구성해야 한다.<br>
- 각 클러스터 kubeconfig 파일 내 cluster, context, user 명이 중복되지 않도록 한다.

```bash
# .kube 디렉터리 생성 및 이동
$ mkdir -p ${HOME}/.kube
$ cd ${HOME}/.kube

# 각 cluster kubeconfig 파일 위치
$ ls ${HOME}/.kube
cluster1-config  cluster2-config

# kubeconfig 파일 경로 설정
$ export KUBECONFIG="${HOME}/.kube/cluster1-config:${HOME}/.kube/cluster2-config"

# context 이름 변경
$ kubectl config rename-context nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4 ncloud
Context "nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4" renamed to "ncloud".
$ kubectl config rename-context nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160 nhn
Context "nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160" renamed to "nhn".
```
##### `NAME`, `CLUSTER`, `AUTHINFO` 값 출력 정상 확인

```bash
$ kubectl config get-contexts
CURRENT   NAME     CLUSTER                                                           AUTHINFO                                                          NAMESPACE
*         ncloud   nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4   nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4   
          nhn      nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160      nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160 
```
##### 클러스터 리소스 조회 정상 확인
```bash
# cluster1 (ncloud) 노드 조회
$ kubectl get nodes --context=ncloud
NAME                      STATUS   ROLES    AGE    VERSION
portal-test-node-w-3f6k   Ready    <none>   3h9m   v1.32.3
portal-test-node-w-3f6l   Ready    <none>   3h9m   v1.32.3
# cluster2 (nhn) 노드 조회
$ kubectl get nodes --context=nhn
NAME                                        STATUS   ROLES    AGE   VERSION
portal-test-cluster-default-worker-node-0   Ready    <none>   38m   v1.32.3

```
<br>

### <span id='4.2'>4.2. StorageClass 설정(참고)
각 클러스터에 `StorageClass`가 설정되어 있는지 확인한다.
> StorageClass를 제공하지 않는다면 별도 설치 필요<br>
> [NFS Server 설치 가이드](../nfs-server-install-guide.md)
```bash
# 클러스터 cluster1의 StorageClass 조회
$ kubectl get storageclass --context=ncloud
NAME                          PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nks-block-storage (default)   blk.csi.ncloud.com   Delete          WaitForFirstConsumer   true                   3h32m
nks-nas-csi                   nas.csi.ncloud.com   Delete          WaitForFirstConsumer   true                   3h32m

#nhn의 경우 storageclass를 제공하지만 자동으로 생성해주지는 않는다.
$ kubectl get storageclass --context=nhn
No resources found
```
nhn storageClass는 별도로 배포해준다.
```yaml
$ vi storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-storageclassi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: cinder.csi.openstack.org
volumeBindingMode: WaitForFirstConsumer
```

<br>

### <span id='4.3'>4.3. Metrics Server 설치(참고)
컨테이너 플랫폼 관리 클러스터의 Metrics 정보 수집을 위해 Metrics Server 설치가 필요하다.
> [Metrics Server 설치](#metrics-server-설치)

```bash
#설치 확인
$ kubectl get pod -A --context=ncloud
NAMESPACE     NAME                                             READY   STATUS    RESTARTS        AGE
kube-system   metrics-server-55b68d74c4-vpn7b                  1/1     Running   0               3h38m

$ kubectl get pod -A --context=nhn
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   metrics-server-75bc4d796b-qjmn8           1/1     Running   0          61m

```

<br>

### <span id='4.4'>4.4. Cilium CNI 사용 클러스터 설정 변경(참고)
- naver cloud<br>

CNI로 `Cilium`을 사용하는 클러스터에서 Istio 구성 시 아래 설정 변경이 필요하다.  
> [Cilium’s integration with Istio](https://docs.cilium.io/en/stable/network/servicemesh/istio) 

#### Cilium Configmap 수정
```bash
$ kubectl edit configmaps cilium-config -n kube-system
```
```yaml
# 추가
bpf-lb-sock-hostns-only: "true"
cni-exclusive: "false"

# 수정
enable-l7-proxy: "false" 
```

#### Cilium DaemonSet 재시작
```bash
# cilium daemonset 재시작
$ kubectl rollout restart daemonset cilium -n kube-system
Warning: spec.template.metadata.annotations[container.apparmor.security.beta.kubernetes.io/mount-cgroup]: deprecated since v1.30; use the "appArmorProfile" field instead
Warning: spec.template.metadata.annotations[container.apparmor.security.beta.kubernetes.io/apply-sysctl-overwrites]: deprecated since v1.30; use the "appArmorProfile" field instead
Warning: spec.template.metadata.annotations[container.apparmor.security.beta.kubernetes.io/clean-cilium-state]: deprecated since v1.30; use the "appArmorProfile" field instead
Warning: spec.template.metadata.annotations[container.apparmor.security.beta.kubernetes.io/cilium-agent]: deprecated since v1.30; use the "appArmorProfile" field instead
daemonset.apps/cilium restarted


# cilium pod 상태 'running' 조회
$ kubectl get pods -n kube-system -l k8s-app=cilium
NAME           READY   STATUS    RESTARTS   AGE
cilium-8sqdf   1/1     Running   0          22s
cilium-pzrb4   1/1     Running   0          22s
```

### <span id='4.5'>4.5. Istio 멀티 클러스터 구성 스크립트 실행

스크립트를 실행하기전에 멀티 클러스터 설정을 위한 변수를 설정해준다.
```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ vi istio-vars-mc.sh 

#이부분 추가
declare -A CLUSTER1_CONFIG
declare -A CLUSTER2_CONFIG
# COMMON VARIABLE (Please change the value of the variables below.)
CLUSTER1_CONFIG[CTX]="ncloud"    # Cluster1 Context Name
CLUSTER2_CONFIG[CTX]="nhn"    # Cluster2 Context Name

# The belows are the default values.
# If you change the values below, there will be a problem with the deploy. Please keep the values.
# TOOL
KUBECTL_VERSION="1.30.3"
HELM_VERSION="3.14.2"
STEP_VERSION="0.24.4"
...

```

Istio 멀티 클러스터 구성을 위한 스크립트를 실행한다.
```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ chmod +x deploy-istio-mc.sh
$ ./deploy-istio-mc.sh
```
```bash
# $ ./deploy-istio-mc.sh 
[Generate Root CA]...
Your certificate has been saved in certs/root-cert.pem.
Your private key has been saved in certs/root-ca.key.
[Install Istio in cluster1]...
Your certificate has been saved in certs/nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4/ca-cert.pem.
Your private key has been saved in certs/nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4/ca-key.pem.
namespace/istio-system created
secret/cacerts created
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Istio core installed ⛵️                                                                                       
✔ Istiod installed 🧠                                                                                          
✔ Installation complete                                                                                        
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Ingress gateways installed 🛬                                                                                
✔ Installation complete                                                                                        
gateway.networking.istio.io/cross-network-gateway created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
[Install Istio in cluster2]...
Your certificate has been saved in certs/nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160/ca-cert.pem.
Your private key has been saved in certs/nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160/ca-key.pem.
namespace/istio-system created
secret/cacerts created
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Istio core installed ⛵️                                                                                       
✔ Istiod installed 🧠                                                                                          
✔ Installation complete                                                                                        
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Ingress gateways installed 🛬                                                                                
✔ Installation complete                                                                                        
gateway.networking.istio.io/cross-network-gateway created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
Error: nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4 is not a valid DNS 1123 label
error: no objects passed to apply
Error: nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160 is not a valid DNS 1123 label
error: no objects passed to apply

--------------------------------------------------------------
[cluster1 (ncloud)] $ istioctl remote-clusters
--------------------------------------------------------------
NAME                                                                SECRET     STATUS     ISTIOD
nks_kr_portal-test-cluster_e50cf226-309b-4c00-8953-a32ff962f8d4                synced     istiod-6d44db4bdc-hlqjs

--------------------------------------------------------------
[cluster2 (nhn)] $ istioctl remote-clusters
--------------------------------------------------------------
NAME                                                             SECRET     STATUS     ISTIOD
nks_portal-test-cluster_7132dd1d-2142-45a4-a7b1-1f0ead2fc160                synced     istiod-87d94786d-8xrq8
```
<br>

#### cluster1 (Ncloud NKS) Istio 리소스 배포 현황 확인
**Ncloud Kubernetes Service**는 기본 CNI로 `Cilium`을 제공한다. <br>
Cilium을 사용하는 Kubernetes 클러스터에서 Istio 사용과 관련된 내용을 참고한다.
> [[Cilium’s integration with Istio]](https://docs.cilium.io/en/stable/network/servicemesh/istio) <br>
> [(참고) [6.3.Cilium CNI 사용 클러스터 설정 변경]](#6.3) 


```bash
$ kubectl get all -n ${ISTIO_NAMESPACE} --context=${CLUSTER1_CONFIG[CTX]}
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-ingressgateway-64cffcd5f9-hphqq   1/1     Running   0          2m45s
pod/istiod-6d44db4bdc-hlqjs                 1/1     Running   0          2m53s

NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                                                                                                      AGE
service/istio-ingressgateway   LoadBalancer   198.19.235.12   istio-syste-istio-ingres-1e9c3-101789248-f924e3285ceb.kr-gov.lb.naverncp.com   15021:31404/TCP,80:31016/TCP,443:31598/TCP,31400:32270/TCP,15443:31181/TCP,15012:31187/TCP,15017:30692/TCP   2m45s
service/istiod                 ClusterIP      198.19.245.3    <none>                                                                         15010/TCP,15012/TCP,443/TCP,15014/TCP                                                                        2m54s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           2m46s
deployment.apps/istiod                 1/1     1            1           2m54s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/istio-ingressgateway-64cffcd5f9   1         1         1       2m46s
replicaset.apps/istiod-6d44db4bdc                 1         1         1       2m54s

NAME                                                       REFERENCE                         TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   cpu: 2%/80%   1         5         1          2m46s

```

#### cluster2 (NHN NKS) Istio 리소스 배포 현황 확인
```bash
$ kubectl get all -n ${ISTIO_NAMESPACE} --context=${CLUSTER2_CONFIG[CTX]}
NAME                                       READY   STATUS    RESTARTS   AGE
pod/istio-ingressgateway-688f8957c-mngvp   1/1     Running   0          2m38s
pod/istiod-87d94786d-8xrq8                 1/1     Running   0          2m44s

NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                                                                                                      AGE
service/istio-ingressgateway   LoadBalancer   10.254.48.228   133.186.217.174   15021:31547/TCP,80:31920/TCP,443:30910/TCP,31400:31847/TCP,15443:32339/TCP,15012:30752/TCP,15017:30897/TCP   2m38s
service/istiod                 ClusterIP      10.254.119.61   <none>            15010/TCP,15012/TCP,443/TCP,15014/TCP                                                                        2m44s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           2m38s
deployment.apps/istiod                 1/1     1            1           2m44s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/istio-ingressgateway-688f8957c   1         1         1       2m38s
replicaset.apps/istiod-87d94786d                 1         1         1       2m44s

NAME                                                       REFERENCE                         TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   cpu: 2%/80%   1         5         1          2m38s
```
<br> 


## <span id='5'>5. 샘플 애플리케이션 배포
멀티 클러스터 통신 테스트를 위해 클러스터 cluster1, cluster2에 샘플 애플리케이션을 배포한다.
> [Istio Bookinfo Application](https://istio.io/latest/docs/examples/bookinfo/)의 일부 애플리케이션 배포

#### 애플리케이션 배포 위치
```bash
# 클러스터 컨텍스트 명 환경변수 설정
$ source ~/workspace/container-platform/cp-portal-deployment/istio_mc/istio-vars-mc.sh

# sample 디렉토리 생성 
$ mkdir -p ~/workspace/container-platform/cp-portal-deployment/istio_mc/sample
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc/sample

# bookinfo 서비스 배포 yaml 다운로드
$ wget https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
# sleep 서비스 배포 yaml 다운로드
$ wget https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
```

<br>

### <span id='5.1'>5.1. 멀티 클러스터 샘플 애플리케이션 배포
```bash
# 각 클러스터에 sample 네임스페이스 생성 및 Istio 사이드카 프록시 자동 주입 활성화
# clusterN에 reviews-N 애플리케이션 배포
$ for IDX in $(seq 1 "$CLUSTER_CNT"); do CTX="CLUSTER${IDX}_CONFIG[CTX]"; \
  echo "context: ${!CTX}"
  kubectl --context=${!CTX} create ns sample; \
  kubectl --context=${!CTX} label namespace sample istio-injection=enabled; \
  kubectl --context=${!CTX} -n sample apply -f bookinfo.yaml -l service=reviews; \
  kubectl --context=${!CTX} -n sample apply -f bookinfo.yaml -l account=reviews; \
  kubectl --context=${!CTX} -n sample apply -f bookinfo.yaml -l app=reviews,version=v${IDX}; \
done
context: ncloud
namespace/sample created
namespace/sample labeled
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
context: nhn
namespace/sample created
namespace/sample labeled
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v2 created
```
```bash
# cluster3에 sleep 애플리케이션 배포
$ kubectl --context="${CLUSTER1_CONFIG[CTX]}" -n sample apply -f sleep.yaml
serviceaccount/sleep created
service/sleep created
deployment.apps/sleep created
```

<br>


### <span id='5.2'>5.2. 멀티 클러스터 통신 테스트
```bash
# cluster1의 reviews-v1, sleep pod 조회
$  kubectl get pods -l app=reviews --context="${CLUSTER1_CONFIG[CTX]}" -n sample
NAME                          READY   STATUS    RESTARTS   AGE
reviews-v1-848b8749df-lwk8r   2/2     Running   0          3m37s

$ kubectl get pods -l app=sleep --context="${CLUSTER1_CONFIG[CTX]}" -n sample
NAME                     READY   STATUS    RESTARTS   AGE
sleep-868c754c4b-s8z6f   2/2     Running   0          103s

# cluster2의 reviews-v2 pod 조회
$ kubectl get pods -l app=reviews --context="${CLUSTER2_CONFIG[CTX]}" -n sample
NAME                          READY   STATUS    RESTARTS   AGE
reviews-v2-5fdf9886c7-qfbvq   2/2     Running   0          3m43s
```
```bash
$ kubectl exec -it -n sample --context="${CLUSTER1_CONFIG[CTX]}" \
"$(kubectl get pod -n sample --context="${CLUSTER1_CONFIG[CTX]}" -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -c sleep -- sh

~ $ curl reviews:9080/reviews/1
{"id": "1","podname": "reviews-v1-848b8749df-fjfcp","clustername": "null","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}]}~ $ 
~ $ curl reviews:9080/reviews/1
{"id": "1","podname": "reviews-v2-5fdf9886c7-mkp4l","clustername": "null","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"error": "Ratings service is currently unavailable"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"error": "Ratings service is currently unavailable"}}]}
```


- 주의할 점
    - 멀티 구성 후 트래픽을 보내기 위해서는 destinationrule, gateway, virtualservice, serviceentry의 배포가 필요하다.
    - destinationrule, gateway, virtualservice, serviceentry 하나라도 배포되지않으면 정상적인 통신 불가능
```yaml
#$ vi destination.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: sample
spec:
  host: reviews.sample.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL

#$ vi gateways.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cluster2-gateway
  namespace: sample
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: PASSTHROUGH
    hosts:
    - "*.sample.svc.cluster.local"

#$ vi virtualservice.yaml 
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-routing
  namespace: sample
spec:
  gateways:
  - cluster2-gateway
  hosts:
  - reviews.sample.svc.cluster.local
  http:
  - match:
    - sourceLabels:
        app: sleep
    route:
    - destination:
        host: reviews.sample.svc.cluster.local
        subset: v1 
      weight: 30
  - match:
    - sourceLabels:
        app: sleep
    route:
    - destination:
        host: reviews.sample.svc.cluster.local
        subset: v2  
      weight: 70 

#$ vi serviceentry.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: reviews-cluster2
  namespace: sample
spec:
  hosts:
  - reviews.sample.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - number: 9080
    name: http
    protocol: HTTP
  resolution: DNS
  addresses:
  endpoints:
  - address: 133.186.217.174
    ports:
      http: 15443 
  exportTo:
  - "."
```



## <span id='6'>6. Istio 멀티 클러스터 구성 삭제
Istio 멀티 클러스터 구성의 삭제를 원하는 경우 아래 스크립트를 실행한다.<br>
:loudspeaker: (주의) 해당 스크립트 실행 시, **Istio 멀티클러스터 구성이 모두 제거**되므로 주의가 필요하다.<br>

```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ chmod +x uninstall-istio-mc.sh
$ ./uninstall-istio-mc.sh
Are you sure you want to uninstall istio in all clusters? <y/n> y # y 입력
```
