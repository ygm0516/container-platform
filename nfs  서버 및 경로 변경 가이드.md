# NFS 서버 및 경로 변경 가이드

- 부득이하게 이미 구축한 운영 환경에서의 NFS 서버와 경로를 변경할 필요가 있을 경우 다음 절차대로 진행

## 목차
1. [nfs-pod-provisioner deployment 변경](#1)
2. [PV 백업](#2)
3. [PV 재생성](#3)


# <div id='1'/> 1. nfs-pod-provisioner deployment 변경

```
$ kubectl edit deployment nfs-pod-provisioner

...
    spec:
      containers:
      - env:
        - name: PROVISIONER_NAME
          value: cp-nfs-provisioner
        - name: NFS_SERVER
          value: {IP 변경}
        - name: NFS_PATH
          value: {경로 변경}
...
      volumes:
      - name: nfs-provisioner
        nfs:
          path: {경로 변경}
          server: {IP 변경}
...

$ kubectl rollout restart deployment nfs-pod-provisioner

```
# <div id='2'/> 2. PV 백업
- 경로를 다르게 설정하였을 경우 경로 수정
```
$ rsync -avz /home/share/nfs {Linux 사용자명}@{서버 IP}:/home/share/nfs
```
# <div id='2'/> 2. PV 재생성
- 기존에 생성된 모든 PV를 대상으로 반복 진행
```
$ kubectl patch pv {PV 이름} -p '{"spec":{"claimRef": null}}'
$ kubectl get pv {PV 이름} -o yaml > {파일명}.yaml
$ kubectl delete pv {PV 이름}

$ vi {파일명}.yaml

...
  nfs:
    path: {경로 수정}/{PV 디렉토리}
    server: {IP 수정}
…

$ kubectl apply -f {파일명}.yaml
```
