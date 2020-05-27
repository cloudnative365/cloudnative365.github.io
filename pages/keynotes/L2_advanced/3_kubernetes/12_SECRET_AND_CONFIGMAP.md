---
title: ConfigMap和Secret
keywords: keynotes, L2_advanced, kubernetes, CKA, SECRET_AND_CONFIGMAP
permalink: keynotes_L2_advanced_3_kubernetes_11_SECRET_AND_CONFIGMAP.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/12_SECRET_AND_CONFIGMAP
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. ConfigMap简介
使用 ConfigMap 可以将我们的配置文件和应用程序代码分开。
比如，假设你正在开发一个应用，它可以在你自己的电脑上（用于开发）和在云上（用于实际流量）运行。你的代码里有一段是用于查看环境变量 DATABASE_HOST，在本地运行时，你将这个变量设置为 localhost，在云上，你将其设置为引用 Kubernetes 集群中的公开数据库 Service 中的组件。
这让我们可以获取在云中运行的容器镜像，并且如果有需要的话，在本地调试完全相同的代码。

## 2. 使用ConfigMap
我们先来通过spec创建一个ConfigMap
``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  Name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: 3
  ui_properties_file_name: "user-interface.properties"
  #
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

我们可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

+ 容器 entrypoint 的命令行参数
+ 容器的环境变量
+ 在只读卷里面添加一个文件，让应用来读取
+ 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap
这些不同的方法适用于不同的数据使用方式。对前三个方法，kubelet 使用 Secret 中的数据在 Pod 中启动容器。

第四种方法意味着你必须编写代码才能读取 Secret 和它的数据。然而，由于您是直接使用 Kubernetes API，因此只要 ConfigMap 发生更改，您的应用就能够通过订阅来获取更新，并且在这样的情况发生的时候做出反应。通过直接进入 Kubernetes API，这个技术也可以让你能够获取到不同的命名空间里的 ConfigMap。

下面是一个 Pod 的示例，它通过使用 game-demo 中的值来配置一个 Pod：
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: game.example/demo-game
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 您可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
```

ConfigMap 不会区分单行属性值和多行类似文件的值，重要的是 Pods 和其他对象如何使用这些值。比如，定义一个卷，并将它作为 /config 文件夹安装到 demo 容器内，并创建四个文件：
+ /config/player_initial_lives
+ /config/ui_properties_file_name
+ /config/game.properties
+ /config/user-interface.properties

如果要确保 `/config` 只包含带有 `.properties` 扩展名的文件，可以使用两个不同的 ConfigMaps，并在 `spec` 中同时引用这两个 ConfigMaps 来创建 Pod。第一个 ConfigMap 定义了 `player_initial_lives` 和 `ui_properties_file_name`，第二个 ConfigMap 定义了 kubelet 放进 `/config` 的文件。

## 3 Secret 简介
Secret 是一种包含有限敏感信息例如密码、token 或 key 的对象。这样的信息可能会被放在 Pod spec 中或者镜像中；将其放在一个 secret 对象中可以更好地控制它的用途，并降低意外暴露的风险。

但是Secret的加密其实是基于base64的编码，而不是某种加密算法，他可以很轻松的通过命令反解。如果想实现真正的加密，必须创建一个带密钥和正确标识的**EncryptionConfiguration**。然后，kube apiserver需要将**--encryption provider config**标志设置为先前配置的加密程序，例如**aescbc**或**ksm**。启用后，还需要重新创建每个secret，因为它们在写入时是加密的。 

用户可以创建 secret，同时系统也创建了一些 secret。要使用 secret，pod 需要引用 secret。Pod 可以用两种方式使用 secret：作为 volume 中的文件被挂载到 pod 中的一个或者多个容器里，或者当 kubelet 为 pod 拉取镜像时使用。

其实我们集群创建完成之后就会有内置的secret，他是给Service Account使用的，他会使用API的证书自动创建和附加secret。

## 4 创建Secret

### 4.1. 使用kubectl创建Secret
假设有些 pod 需要访问数据库。这些 pod 需要使用的用户名和密码在本地的 ./username.txt 和 ./password.txt 文件里。
``` bash
# Create files needed for rest of example.
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
```
使用`kubectl create secret`命令将这些文件打包到一个 Secret 中并在 API server 中创建一个对象。
``` bash
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
secret "db-user-pass" created
```
我们可以这样检查刚创建的 secret：
``` bash
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s
```

``` bash
kubectl describe secrets/db-user-pass
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

默认情况下，`kubectl get` 和 `kubectl describe` 避免显示密码的内容。 这是为了防止机密被意外地暴露给旁观者或存储在终端日志中。

### 4.2. 手动创建 Secret
我们也可以先以 json 或 yaml 格式在文件中创建一个 secret 对象，然后创建该对象。 密码包含两种类型，数据和字符串数据。 数据字段用于存储使用 base64 编码的任意数据。 提供 stringData 字段是为了方便起见，它允许您将机密数据作为未编码的字符串提供。

例如，要使用数据字段将两个字符串存储在 Secret 中，请按如下所示将它们转换为 base64
``` bash
echo -n 'admin' | base64
YWRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```
现在可以像这样写一个 secret 对象
``` bash
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```
使用 kubectl apply 创建 secret
``` bash
kubectl apply -f ./secret.yaml
secret "mysecret" created
```

## 5. 使用Secret
Secret 可以作为数据卷被挂载，或作为环境变量 暴露出来以供 pod 中的容器使用。它们也可以被系统的其他部分使用，而不直接暴露在 pod 内。 例如，它们可以保存凭据，系统的其他部分应该用它来代表您与外部系统进行交互。

### 5.1. 在 Pod 中使用 Secret 文件
#### 在 Pod 中的 volume 里使用 Secret：

* 创建一个 secret 或者使用已有的 secret。多个 pod 可以引用同一个 secret。
* 修改您的 pod 的定义在 spec.volumes[] 下增加一个 volume。可以给这个 volume 随意命名，它的 spec.volumes[].secret.secretName 必须等于 secret 对象的名字。
* 将 spec.containers[].volumeMounts[] 加到需要用到该 secret 的容器中。指定 spec.containers[].volumeMounts[].readOnly = true 和 spec.containers[].volumeMounts[].mountPath 为您想要该 secret 出现的尚未使用的目录。
* 修改您的镜像并且／或者命令行让程序从该目录下寻找文件。Secret 的 data 映射中的每一个键都成为了 mountPath 下的一个文件名。
这是一个在 pod 中使用 volume 挂在 secret 的例子：
``` bash
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

每个 secret 都需要在 spec.volumes 中指明。
如果 pod 中有多个容器，每个容器都需要自己的 volumeMounts 配置块，但是每个 secret 只需要一个 spec.volumes。
可以打包多个文件到一个 secret 中，或者使用的多个 secret，怎样方便就怎样来。

#### 查看结果
在挂载的 secret volume 的容器内，secret key 将作为文件，并且 secret 的值使用 base-64 解码并存储在这些文件中。这是在上面的示例容器内执行的命令的结果：
``` bash
ls /etc/foo/
username
password

cat /etc/foo/username
admin

cat /etc/foo/password
1f2d1e2e67df
```

### 5.2. Secret 作为环境变量
#### 将 secret 作为 pod 中的环境变量使用：

* 创建一个 secret 或者使用一个已存在的 secret。多个 pod 可以引用同一个 secret。
* 修改 Pod 定义，为每个要使用 secret 的容器添加对应 secret key 的环境变量。消费secret key 的环境变量应填充 secret 的名称，并键入 env[x].valueFrom.secretKeyRef。
* 修改镜像并／或者命令行，以便程序在指定的环境变量中查找值。
这是一个使用 Secret 作为环境变量的示例：
``` bash
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```
#### 查看结果
在一个消耗环境变量 secret 的容器中，secret key 作为包含 secret 数据的 base-64 解码值的常规环境变量。这是从上面的示例在容器内执行的命令的结果
``` bash
echo $SECRET_USERNAME
admin
echo $SECRET_PASSWORD
1f2d1e2e67df
```