## Worker node 추가 및 삭제 가이드.md


## 목차
1. [Worker 노드 추가](#1)
    * [1.1. kube-apiserver.yaml 수정](#1-1)
    * [1.2. 추가할 Worker Node에 공개키 입력](#1-2)
    * [1.3. hosts.yaml 수정](#1-3)
    * [1.4. ansible-playbook 명령어 실행](#1-4)
    * [1.5. 정상 작동 확인](#1-5)
    * [1.6. NFS 스토리지 사용 시 설정 추가](#1-6)

2. [Worker 노드 삭제](#2)
    * [2.1. Worker 노드 삭제](#2-1)

3. [동일 IP Worker node 추가](#3)
    * [3.1. Worker node 클러스터 설정 초기화](#3-1)
    * [3.2. Worker node 추가 진행](#3-2)
    


# <div id='1'/> 1. Worker 노드 추가
## <div id='1-1'/> 1.1. kube-apiserver.yaml 수정
```
$ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
...
    - --anonymous-auth=True  #True로 수정
...
```

- 수정하면 api-server가 재시작 되므로 아래와 같이 에러 발생(몇초 후 다시 조회하면 정상화 됨)
```
$ kubectl get nodes
The connection to the server xx.xx.xx.xx:6443 was refused - did you specify the right host or port?
```

## <div id='1-2'/> 1.2. 추가할 Worker Node에 공개키 입력

```
#Master Node에서 실행
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5QrbqzV6g4i....

#추가할 Worker Node에서 실행
$ vi .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDRueywSiuwyf...
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5QrbqzV6g4i... #추가(Master Node 공개키 값 추가)
```
## <div id='1-3'/> 1.3. hosts.yaml 수정
```
# Master Node에서 실행
# hosts.yaml 파일에 새로 추가할 Worker Node에 대한 정보 입력(hostname과 ip는 예시이므로 해당 서버정보 값 입력)
$ sudo vim ~/cp-deployment/standalone/inventory/mycluster/hosts.yaml
all:
  hosts:
    cluster1:
      ansible_host: 192.168.0.xx
      ip: 192.168.0.xx
      access_ip: 192.168.0.xx
    cluster2:
      ansible_host: 192.168.0.xx
      ip: 192.168.0.xx
      access_ip: 192.168.0.xx
    cluster3:                                 #추가(새로 추가할 Worker Node Host name)
      ansible_host: 192.168.0.xx              #추가(새로 추가할 Worker Node Private IP)
      ip: 192.168.0.xx                        #추가(새로 추가할 Worker Node Private IP)
      access_ip: 192.168.0.xx                 #추가(새로 추가할 Worker Node Private IP)
  children:
    kube_control_plane:
      hosts:
        cp-test-cluster1:
    kube_node:
      hosts:
        cluster1:
        cluster2:
        cluster3:                              #추가(새로 추가할 Worker Node Host name)
    etcd:
      hosts:
        cp-test-cluster1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
## <div id='1-4'/> 1.4. ansible-playbook 명령어 실행
```
$ cd ~/cp-deployment/standalone
$ ansible-playbook -i ~/cp-deployment/standalone/inventory/mycluster/hosts.yaml --become --become-user=root scale.yml
```
## <div id='1-5'/> 1.5. 정상 작동 확인
```
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS        AGE
nfs-pod-provisioner-57bf7fdc86-bktsz   1/1     Running   5 (6m40s ago)   65m


$ kubectl get pods -n kube-system
NAME                                           READY   STATUS    RESTARTS        AGE
calico-kube-controllers-648dffd99-xxp2n        1/1     Running   0               68m
calico-node-2tq6v                              1/1     Running   0               67s
calico-node-mqn2b                              1/1     Running   0               68m
calico-node-zg99s                              1/1     Running   0               68m
coredns-77f7cc69db-8ps6j                       1/1     Running   0               68m
coredns-77f7cc69db-n54qt                       1/1     Running   0               68m
dns-autoscaler-8576bb9f5b-dm78l                1/1     Running   0               68m
kube-apiserver-cp-test-cluster1            1/1     Running   0               6m36s
kube-controller-manager-cp-test-cluster1   1/1     Running   4 (6m59s ago)   69m
kube-proxy-9v6fp                               1/1     Running   0               64s
kube-proxy-qx9dz                               1/1     Running   0               65s
kube-proxy-rf8r5                               1/1     Running   0               65s
kube-scheduler-cp-test-cluster1            1/1     Running   4 (6m59s ago)   69m
metrics-server-bd6df7764-cb5zm                 1/1     Running   0               67m
nginx-proxy-cp-test-cluster2               1/1     Running   0               69m
nginx-proxy-cp-test-cluster3               1/1     Running   0               60s
nodelocaldns-7h4jt                             1/1     Running   0               68m
nodelocaldns-f4w7f                             1/1     Running   0               68m
nodelocaldns-ht6bj                             1/1     Running   0               67s

$ kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
cluster1   Ready    control-plane   69m   v1.28.6
cluster2   Ready    <none>          69m   v1.28.6
cluster3   Ready    <none>          70s   v1.28.6     #Worker Node 추가 확인

```
## <div id='1-6'/> 1.6. NFS 스토리지 사용 시 설정 추가

NFS 스토리지를 사용하신 경우, 신규 Worker 노드에 대한 설정을 추가 <br/>
https://github.com/K-PaaS/container-platform/blob/master/install-guide/nfs-server-install-guide.md

# <div id='2'/> 2. Worker 노드 삭제
## <div id='2-1'/> 2.1. kube-apiserver.yaml 수정

노드를 스케줄러에서 제외시켜 파드가 할당되지 않도록 하거나 기존에 배포된 파드를 다른 노드로 이동시키는 drain을 적용하여 노드의 자원을 안전하게 이동시켜야함

drain 작업 이후에 다음 노드 삭제를 진행
```
$ kubectl get nodes
$ kubectl drain --ignore-daemonsets [NODE_NAME]
$ kubectl delete node [NODE_NAME]
```

# <div id='3'/> 3. 동일 IP Worker node 추가
## <div id='3-1'/> 3.1. Worker node 클러스터 설정 초기화

```
$ sudo su
# kubeadm reset 
# rm -rf ~/.config/cni/net.d
# rm -rf /etc/kubernetes/* 
# rm -rf /var/lib/kubelet
```
## <div id='3-2'/> 3.2. Worker node 추가 진행
<a href="#1">Worker node 추가 가이드</a>
