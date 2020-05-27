## 概念

`PersistentVolume`(PV)

为了让业务和存储分离,所以抽象出了PV,存储配置好,用PV挂载到存储上即可,比如nfs,比如某某云



`PersistentVolumeClaim`(PVC)

PV的请求,需要绑定,才可饮用



绑定

PVC和PV是一对一,但是Pord可以多个共享一个PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv0003
spec:
    capacity:
        storage: 5Gi
    volumeModie: Filesystem
    accessModes:
    - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    storageClassName: slow
    mountOptions:
    - hard
    - nffsvers=4.1
    nfs: 
        path: /tmp
        server: 172.17.0.2
```



**演示**

1. 安装NFS服务

   ```shell
   yum install -y nfds-common nfs-utils rpcbind
   mkdir /nfsdata
   chmod 666 /nfsdata
   chown nfsnobody /nfsdata
   cat /etc/exports
       /nfsdata *(rw,no_root_squash,no_all_squash,sync)
   systemctl start rpcbind
   systemctl start nfs
   ```

2. 部署PV

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
      name: nfdspv1
   spec:
      capacity:
         storage: 1Gi
      accessModes:   # 通常pvc会根据 accessMode   storageClassName  storeage来判断绑定哪个pv
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Recycle
      storageClassName: nfs   # 这就是一个标签,标示这个pv的属性比如快,比如大等特性
      nfs:
          path: /data/nfs
          server: 10.66.66.10    #具体是哪个机器上的nfs
   ```

   

3. 创建服务并使用pvc

```yaml
apiVersion: v1
kind: Service
metadata:
    name: nginx
    labels:
        app:nginx
spec:
    ports:
    - ports: 80
      name: web
      clusterIp: None   #这个无头服务会给statfulset域名访问的能力
      selector:
          app:nginx
 ---
 apiVersion: v1
 kind: StatefulSet   # 有状态存储
 metdata:
     name: web
 spec:
     selector:
         matchLabels:
             app: nginx
     servicename: "nginx"   #匹配无头服务的名字
     replicas: 3
     template:
         metadata:
             labels:
                 app: nginx
         spec:
             containers:
             - name: nginx
             image: nginx
             ports:
             - containerPorts: 80
               name: web
             volumeMounts:   
             - name: www
               mountPath: /usr/share/nginx/html  # 挂载的存储是本地这个路径
     volumeClaimTemplates:     # 这个就是PVC,就是用他来绑定PV
     - metadata:
         name: wwww
       spec:   # 这个PVC有三个要求, accessMode是RWO,storageClassName是nfs,store:1G
           accessModes: ["ReadWriteOnce"]
           storageClassName: "nfs"
           resources:
               request:
                   store: 1Gi
```

