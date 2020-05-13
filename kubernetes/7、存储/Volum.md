# Volum

容器磁盘上的文件的生命周期是短暂的,这就使得在容器中运行重要应用时会出现一些问题.首先,当容器崩溃时,kubelet会**重启**它,但是容器中的文件将丢失--容器以**干净的状态**(镜像最初的状态)重新启动.其次,在Pod中同时运行多个容器时,这些容器之间通常需要**共享文件**.kubernetes中的Volume抽象就很好的解决了这些问题

## 背景

Kubernetes中的卷有明确的寿命--与封装它的Pod相同.所以,卷的生命比Pod中的所有容器都长,当这个容器重启时数据仍然得以保存.当然,当Pod不再存在时,卷也将不复存在.也许更重要的是,Kubernetes支持多种类型的卷,Pod可以同时使用任意数量的卷



## 卷的类型

Kubernetes支持以下类型的卷:

awsElasticBlockStore、azureDisk、azureFile、cephfs、csi、downwardAPI、emptyDir、fc、flocker、gcePersistentDisk、gitRepo、glusterfs、hostPath、iscsi、local、nfs、persistentVolumeClaim、projected、portworxVolume、quobyte、rbd、scaleIO、secret、storeageos、vsphereVolume



### emptyDir

当Pod被分配给节点时,首先创建emptyDir卷,并且只要该Pod在该节点上运行,该卷就会存在,正如卷的名字所述,它最初时空的.Pod中的容器可以读取和写入emptyDir卷中的相同文件,尽管该卷可以挂载到每个容器中的相同或不同路径上.当处于任何原因从节点中删除Pod时,emptyDir中的数据将被永久删除

emptyDir的用法有:

- 暂存空间,例如用于给予磁盘的合并排序
- 用作长时间计算崩溃回复时的检查点
- web服务器容器提供数据时,保存内容管理容器提取的文件

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: test-pd
spec:
    containers:
    - image: test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /cache
        name: cache-volume
    volumes:
    - names: cache-volume
      emptyDir:{}
```

### hostPath

`hostPath`卷将主机节点的文件系统中的文件或目录挂载到集群中

`hostPath`的用途如下:

- 运行需要访问Docker内部的容器;使用`/var/lib/docker`的`hostPath`
- 在容器中运行cAdvisor;使用/dev/cgroups的hostPath
- 允许pod指定给定的hostPath是否应该在pod运行之前存在,是否应该创建,以及它应该以什么形式存在

除了所需的path属性之外,用户还可以为hostPath卷指定type

| 值                | 行为                                                         |
| ----------------- | ------------------------------------------------------------ |
|                   | 空字符串(默认)用于向后兼容,这意味着在挂载hostPath卷之前不会执行任何检查 |
| DirectoryOrCreate | 如果在给定的路径上没有任何东西存在,那么将根据需要在那里创建一个空目录,权限设置为0755,与Kubelet具有相同的组和所有权. |
| Directory         | 给定的路径下必须存在目录                                     |
| FileOrCreate      | 如果在给定的路径上没有任何东西存在,那么会根据需要创建一个空文件,权限设置为0644,与Kubelet具有相同的组和所有权 |
| File              | 给定的路径下必须存在文件                                     |
| Socket            | 给定的路径下必须存在Unix套接字                               |
| CharDevice        | 给定的路径下必须存在字符设备                                 |
| BlockDevice       | 给定的路径下必须存在块设备                                   |

使用这种卷类型要注意,因为:

由于每个节点上的文件都不同,具有相同配置