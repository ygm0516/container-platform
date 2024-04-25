### [Index](https://github.com/K-PaaS/container-platform/blob/playpark/README.md) > [CP Use](../Readme.md) >  플레이 파크 포털 사용 가이드

<br>

## Table of Contents

1. [문서 개요](#1)
   * [1.1. 목적](#1-1)
   * [1.2. 범위](#1-2)
2. [플레이 파크](#2)
   * [2.1. 쿠버네티스 클러스터 설정](#2-1)
3. [플레이 파크 컨테이너 플랫폼 포털을 활용한 MySQL과 WordPress 배포](#3)
   * [3.1. portal을 활용한 mysql 배포](#3-1)
   * [3.1.1. persistenctVolume 생성](#3-1-1)
   * [3.1.2. persistenctVolumeClaim 생성](#3-1-2)
   * [3.1.3. MySQL deployment 생성](#3-1-3)
   * [3.1.4. MySQL service 생성](#3-1-4)
   * [3.2. portal을 활용한 WordPress 배포](#3-2)
   * [3.2.1. persistenctVolume 배포](#3-2-1)
   * [3.2.2. persistenctVolumeClaim 배포](#3-2-2)
   * [3.2.3. WordPress deployment 생성](#3-2-3)
   * [3.2.4. WordPress service 생성](#3-2-4)

4. [Container-platform portal을 활용한 MySQL과 WordPress 삭제](#4)
   * [4.1. 삭제](#4-1)

<br>
# <div id='1'/> 1. 문서 개요

## <div id='1-1'/> 1.1. 목적
본 문서는 플레이 파크 컨테이너 플랫폼 포털을 활용하여 WordPress 사이트와 MySQL 데이터베이스를 어떻게 배포하는지 보여준다.


<br>

## <div id='1-2'/> 1.2. 범위
본 문서는 플레이파크 단독형 배포를 기준으로 작성되었다.
플레이 파크 단독형 배포를 기준으로 컨테이너 플랫폼 포털을 활용하여 WordPress와 MySQL을 배포하는 방법을 기술하였다.

<br>
# <div id='2'/>2. 플레이 파크

## <div id='2-1'/>2.1. 플레이파크 접속
플레이 파크에 접속해서 로그인을 한다. 
로그인 후 cp portal 대시보드에 접속한다.
<br>


# <div id='3'/>3. 플레이 파크 컨테이너 플랫폼 포털을 활용한 MySQL과 WordPress 배포

## <div id='3-1'/>3.1. 플레이 파크 컨테이너 플랫폼 포털을 활용한 MySQL 배포
본 장에서는 플레이파크 컨테이너 플랫폼 포털을 활용하여 MySQL의 배포 방법에 대해 기술하였다.

### <div id='3-1-1'/> 3.2.1. persistentVolume 생성
MySQL은 각 데이터를 저장할 퍼시스턴트볼륨이 필요하다.
많은 클러스터 환경에서 설치된 기본 스토리지클래스가있는데 다음 코드는 기본 스토리지클래스를 사용하여 퍼시스턴트볼륨을 생성하는 코드이다.

이미지 사진!

'''

kind: PersistentVolume
apiVersion: v1
metadata:
  name: mysql-pv
  lables:
    app: wordpress
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/k8s/mysql"
  storageClassName: cp-storageclass

 '''

### <div id='3-1-2'/> 3.1.2. persistentVolumeClaim 생성

	이미지 사진!

'''

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-pvc
  lables:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

 '''


### <div id='3-1-3'/> 3.1.3. MySQL deployment 생성


'''

apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc

'''



### <div id='3-1-4'/> 3.1.4. MySQL service 생성

'''
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
      protocol: TCP
      nodePort: 32009
  selector:
    app: wordpress
    tier: mysql
  type: NodePort

'''


## <div id='3-2'/>3.2. 플레이 파크 컨테이너 플랫폼 포털을 활용한 WordPress 배포

### <div id='3-2-1'/> 3.2.1. persistentVolume 생성
'''

kind: PersistentVolume
apiVersion: v1
metadata:
  name: wp-pv
  lables:
    app: wordpress
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/k8s/wp"
  storageClassName: cp-storageclass
  
'''

### <div id='3-2-2'/> 3.2.2. persistentVolumeClaim 생성

'''

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pvc
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

'''

### <div id='3-2-3'/> 3.2.3. WordPress Deployment 생성

'''
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: root
        - name: WORDPRESS_DB_PASSWORD
          value: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pvc

'''

### <div id='3-2-4'/> 3.2.4. WordPress Service 생성


'''

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
      NodePort: 33009
  type: NodePort
  selector:
    app: wordpress
    tier: frontend

'''

