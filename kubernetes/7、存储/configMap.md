# configMap

## configMap描述信息

ConfigMap功能在Kubernetes1.2版本中引入,许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息.ConfigMap API 给我们提供了向容器中注入配置信息的机制,ConfigMap可以被用来保护单个属性,也可以用来保存整个配置文件或者JSON二进制大对象

## ConfigMap的创建

1. 使用目录创建

   ```shell $ls docs/user-guide/configmap/kubectl/
   $ls docs/user-guide/configmap/kubectl/
   
   game.properties
   ui.properties
   ```

   ```shell
   $cat docs/user-guide/configmap/kubectl/game.properties
   
   enemies=aliens
   lives=3
   enemies.cheat=true
   enemies.cheat.level=noGoodRotten
   secret.code.passphrase=ADF
   
   $ cat docs/user-guide/configmap/kubectl/ui.properties
   color.good=purple
   color.bad=yellow
   allow.textmode=true
   how.nice.to.look=fairlyNice
   ```

   ```shell
   $ kubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl
   # --from-file 指定在目录下的所有文件都会被用在ConfigMap里面创建一个K/V,K的名字就是文件名,V就是文件内容
   ```

2. 使用文件创建

   只要指定为一个文件就可以从单个文件中创建ConfigMap

   ```shell
   $ kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties
   
   $kubectl get configmaps game-config-2 -o yaml
   #--from-file这个参数可以使用多次,你可以使用两次分别指定上个实例中的那两个配置文件,效果就跟指定整个目录一样
   ```

   

3. 使用字面值创建

   使用文字值创建,利用--from-literal参数传递配置信息,该参数可以使用多次,格式如下

   ```shell
   $kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
   
   $kubectl get configmaps special-config -o yaml
   ```



## Pod中使用ConfigMap

1. 使用ConfigMap来替代环境变量

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
       name: special-config
       namespace: default
   data:
       special.how: very
       special.type: charm
   ```

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
       name: env-config
       namespace: default
   data:
       log_level: INFO
   ```

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata: 
       name: dapi-test-pod
   spec:
       containers:
       - name: test-container
         image: myapp:v1
         command: ["/bin/sh","-c","env"]
         env:
         - name: SPECIAL_LEVEL_KEY
           valueFrom:
               name: special-config
               key: special.how
         - name: SPECIAL_TYPE_KEY
           valueFrom:
               name: special-config
               key: special.type
         envFrom: 
         - configMapRef:
               name: env-config
   restartPolicy: Never
   ```

2. 用ConfigMap设置命令行参数

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
       name: special-config
       namespace: default
   data:
       special.how: very
       special.type: charm
   ```

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
       name: dapi-test-pod
   spec:
       containers:
       - name: test-container
         image: myapp.v1
         command: ["/bin/sh","-c","echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)"]
         env:
         - name: SPECIAL_LEVEL_KEY
           valueFrom:
           configMapKeyRef:
               name: special-config
               key: special.how
         - name: SPECIAL_TYPE_KEY
           valueFrom:
           configMapKeyRef:
               name: special-config
               key: special.type
      restartPoicy: Never
   ```

3. 通过数据卷插件使用ConfigMap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
       name: special-config
       namespace: default
   data:
       special.how: very
       special.type: charm
   ```

   在数据卷里面使用这个ConfigMap,有不同的选项.最基本的就是将文件填入数据卷,在这个文件中,K就是文件名,V就是文件内容

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
       name: dapi-test-pod
       namespace: default
   spec: 
       container:
       - name: test-container
         image: myapp:v1
         command: ["/bin/sh", "-c", "cat /etc/config/special.how"]
         volumeMounts:
         - name: config-volume
           mountPath: /etc/config
       volumes:
       - name: config-volume
         configMap: 
             name: special-config
       restartPolicy: Never
   ```



## ConfigMap的热更新

```yaml
 apiVersion: v1
kind: ConfigMap
meta:
    name: log-config
    namespace: default
data:
    log_level: INFO
---
apiVersoin: v1
kind: Deployment
metadata:
    name: my-nginx
spec:
    replicas: 1
    template:
    metadata:
        labels:
            run: my-nginx
    spec:
        containers:
        - name: my-nginx
          image: myapp:v1
          posts:
          - containerPort: 80
          volumeMounts:
          - name: config-volume
            mountPath: /et/config
        volumes:
        - name: config-volume
          configMap:
              name: log-config
```

```shell
$kubectl exec `kubectl get pods -l run=my-nginx -o=name|cut -d "/" -f2` cat /etc/config/log_level
INFO
```

#### ConfigMap更新后滚动更新Pod

更新ConfigMap目前并不会触发相关Pod的滚动更新,可以通过修改pod annotations的方式强制触发滚动更新

```shell
$kubectl patch deployment my-nginx --patch '{"spec":{"template":{"metadata":{"annotations":{"version/config":}}}}}'
```

这个例子里我们在`.spec.template.metadata.annotations`中添加 `version/config`,每次通过修改 `version/config`来触发滚动更新

!!!更新ConfigMap后:

- 使用该ConfigMap挂载的Env
- 使用该ConfigMap挂载的Volume中的数据需要一段时间(实测需要10s)才能更新













































