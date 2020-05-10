# Secret存在意义

Secret解决了秘密、token、密钥等敏感数据的配置问题,而不需要把这些敏感数据暴露到镜像或者Pod Sec中.Secret可以以Volumn或者环境变量的方式使用

##### Secret有三种类型:

- **Servcie Account:**用来访问KubernetesAPI,由Kubernetes自动创建,并且会自动挂载到Pod的`/run/secrets/kubernetes.io/serviceaccount`目录中
- **Opaque**:base64编码格式的Secret,用来存储密码、密钥等
- **kubernetes.io/dockerconfigjson**:用来存储私有docker registry的认证信息



## Service Account

Service Account用来访问Kunbernetes API,由Kubernetes自动创建,并且会自动挂载到Pod的`/run/secrets/kubernetes.io/serviceaccount`目录中

```shell
$kubectl run nginx --image nginx
deployment "nginx" created
$kubectl get pods

```

| NAME            | READY | STATUS  | RESTARTS | AGE  |
| --------------- | ----- | ------- | -------- | ---- |
| nginx-331-md1u2 | 1/1   | Running | 0        | 13s  |

```shell
$kubectl exec nginx-331-md1u2 ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

## 

1. 创建说明

   Opaque类型的数据是一个map类型,要求value是base64编码格式:

   ```shell
   $echo -n "admin" | base64
   YWRtaW4=
   $echo -n "1f2d1e2e67df" | base64
   MWYyZDFlMmU2N2Rm
   ```

   secrets.yml

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
       name: mysecret
   type: Opaque
   data:
       password: MWYyZDFlMmU2N2Rm
       username: YWRtaW4=
   ```

   

2. 使用方式

   将Secret挂载到Volume中

```yaml
apiVersion: v1
kind: Pod
metadata:
    labels:
        name:seret-test
    name: seret-test
spec:
    volumes:
    - name: secrets
      secret:
          secretName:mysecret
    containers:
    - image: myapp:v1
      name: db
      volumeMounts:
      - name: secrets
        mountPath: ''
        readOnly: true
```

将Secret导出到环境变量中

```yaml
apiVersion: v1
kind: Deployment
metadata:
    name: pod-deployment
sepc:
    replicas: 2
    template: 
        metadata: 
            labels:
                app: pod-deployment
        spec:
            containers:
            - name: pod-1
              image: myapp:v1
              ports:
              - containerPort: 80
              env:
              - name: TEST_USER
                valueFrom:
                    secretKeyRef:
                        name: mysecret
                        key: password
```



## Kubernetes.io/dockerconfigjson

使用kuberctl创建docker registry 认证的secret

```shell
$kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER+EMAIL
```

在创建Pod的时候,通过`imagePullSecrets`来引用刚创建的`myregistrykey`

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: foo
spec: 
    containers:
    - name: foo
      image: myapp:v1
    imagePullSecrets:
    - name: myregistrykey
```

































