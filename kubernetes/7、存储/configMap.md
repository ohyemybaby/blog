# configMap

## configMap描述信息

ConfigMap功能在Kubernetes1.2版本中引入,许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息.ConfigMap API 给我们提供了向容器中注入配置信息的机制,ConfigMap可以被用来保护单个属性,也可以用来保存整个配置文件或者JSON二进制大对象

## ConfigMap的创建

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
```



```

```