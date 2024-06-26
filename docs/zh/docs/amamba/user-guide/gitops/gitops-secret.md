# GitOps Secret 加密

在 GitOps 运维模式中，待部署资源都是以 yaml 的形式存放在 Git 仓库中，而这些文件中可能包含敏感信息，例如数据库密码、API 密钥等，它们不应该以明文的形式存储，
同时，当这些资源被部署在 kubernetes 集群中时，即使是以 secret 的方式，依旧可以通过 base64 很轻易的查看，这会造成很多安全问题。

为了解决这些问题，本文将介绍以下几种方案来实现 GitOps 中 manifest 文件的加密功能。 实现方案主要分为两类：

1. 基于 argocd 的 plugin 机制，在渲染 manifest 文件时进行解密并替换敏感信息

   这种方式的优点是：

    - 与argocd结合紧密,不需要安装额外的组件
    - 可以方便的与现有的凭证管理系统集成
    - 支持很多的凭证存储后端，如vault、k8s secret、aws secret等
    - 可以支持任意kubernetes资源的加密,如secret,configmap, deployment的环境变量等

   缺点：敏感信息变更后需要 **手动同步**

2. 不依赖于argocd，依赖于项目或工具本身的加解密能力对敏感信息进行加解密

   这种方式的优点是：

    - 不依赖与具体的GitOps实现
    - 安全性更强（如无法轻易解密，或无法通过kubectl describe等方式查看）
    - 配置简单
    - 完全的GitOps体验，不需要手动同步

   缺点：需要单独安装工具，需要手动配置

请根据实际的使用情况选择合适的方案。

## 基于argocd的plugin机制

此方案中使用到的是[argocd-vault-plugin](https://github.com/argoproj-labs/argocd-vault-plugin)插件，它将会从指定的后端存储中获取敏感信息，可以与argocd较好的结合。

它支持的后端储存很多，本文以**HashiCorp Vault**和**kubernetes secret**为例。
> 如何使用vault不是本文的重点，这里只介绍如何配置此插件使用vault作为后端敏感信息的存储。

在使用之前需要在安装argocd时进行一些配置，具体步骤如下：

### 安装配置

#### 管理员设置后端存储访问密钥

安装argocd 之前，需要管理员先创建一下secret，用于此插件访问后端存储

1. 后端存储为HashiCorp Vault

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: argocd-vault-plugin-credentials
      namespace: argo-cd                               # argocd部署的命名空间  
    data:
      AVP_TYPE: vault                                  # 指定后端存储的类型vault
      AVP_AUTH_TYPE: token                             # 指定后端存储的auth类型
      VAULT_ADDR: 10.6.10.11                           # valut的地址
      VAULT_TOKEN: cm9vdA==                            # 通过vault的pod日志获取到的初始化token（在实际使用时，需要设置为具体的权限和访问策略获取到的token）
    type: Opaque
    ```

   通过配置上述secret，在argocd运行期间，会将data中的配置项以环境变量的形势传递给`argocd-vault-plugin`，用于此插件访问具体的vault来获取敏感数据。

2. 后端存储为 kubernetes secret

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: argocd-vault-plugin-credentials
      namespace: argo-cd
    data:
      AVP_TYPE: kubernetessecret    # 设置后端存储类型为 kubernetessecret
    type: Opaque
    ```

上面为最基础的配置，如果后端存储为其他类型, 或者需要有其他的配置项请查看[后端存储配置](https://argocd-vault-plugin.readthedocs.io/en/stable/backends/)。

#### 安装argocd

应用工作台中默认安装了argoCD, 也可以单独安装，具体的安装步骤请看[安装argocd](../../pluggable-components.md), 下面根据两种情况进行配置。

##### 修改工作台默认安装的argoCD

1. 添加一个configmap,用于配置插件

    前往 容器管理 -> 集群列表选择 kpanda-global-cluster -> 配置与密钥 -> 通过yaml新建， 内容如下:
   
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: cmp-plugin
      namespace: argo-cd
    data:
      avp.yaml: |
        apiVersion: argoproj.io/v1alpha1
        kind: ConfigManagementPlugin
        metadata:
          name: argocd-vault-plugin
        spec:
          allowConcurrency: true
          discover:
            find:
              command:
                - sh
                - "-c"
                - "find . -name '*.yaml' | xargs -I {} grep \"<path\\|avp\\.kubernetes\\.io\" {} | grep ."
          generate:
            command:
              - argocd-vault-plugin 
              - generate 
              - --verbose-sensitive-output=true 
              - ./
          lockRepo: false
      avp-helm.yaml:
        apiVersion: argoproj.io/v1alpha1
        kind: ConfigManagementPlugin
        metadata:
          name: argocd-vault-plugin-helm
        spec:
          allowConcurrency: true
          discover:
            find:
              command:
                - sh
                - "-c"
                - "find . -name 'Chart.yaml' && find . -name 'values.yaml'"
          generate:
            command:
                - sh
                - "-c"
                - |
                  helm template $argocd_APP_NAME -n $ARGOCD_APP_NAMESPACE ${argocd_ENV_HELM_ARGS} . |
                  argocd-vault-plugin generate -
          lockRepo: false 
    ```

2. 修改argocd-repo-server的deployment
   前往 容器管理 -> 集群列表选择 kpanda-global-cluster -> 工作负载 -> 无状态负载 -> 选择命名空间 argocd -> 编辑 argocd-repo-server 的yaml，进行如下修改：

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: argocd-repo-server
      namespace: argo-cd
    spec:
      template:
        spec:
          volumes:                                                             # 添加volumes
            - name: cmp-plugin 
              configMap:
                name: cmp-plugin
                defaultMode: 420
            - name: custom-tools
              emptyDir: {}
          initContainers:
            - name: init-vault-plugin                                          # 添加一个新的initContainer
              image: release.daocloud.io/amamba/argocd-vault-plugin:v1.17.0    # 如果是离线环境需要替换地址
              command:
                - sh
                - '-c'
              args:
                - cp /usr/local/bin/argocd-vault-plugin /custom-tools
              volumeMounts:
                - name: custom-tools
                  mountPath: /custom-tools
          containers:
            - name: avp                                                       # 添加一个sidecar容器
              image: quay.io/argoproj/argocd:v2.10.4                          # 镜像地址需要与repo-server的相同
              command:
                - /var/run/argocd/argocd-cmp-server
              envFrom:
                - secretRef:
                  name: argocd-vault-plugin-credentials
              volumeMounts:
                - name: var-files
                  mountPath: /var/run/argocd
                - name: plugins
                  mountPath: /home/argocd/cmp-server/plugins
                - name: tmp
                  mountPath: /tmp
                - name: cmp-plugin
                  mountPath: /home/argocd/cmp-server/config/plugin.yaml
                  subPath: avp.yaml
                - name: custom-tools
                  mountPath: /usr/local/bin/argocd-vault-plugin
                  subPath: argocd-vault-plugin
            - name: repo-server                                              # 修改原有的repo-server container，添加envFrom和volumeMounts
              envFrom:
                - secretRef:
                    name: argocd-vault-plugin-credentials
              volumeMounts:
                - name: cmp-plugin
                  mountPath: /home/argocd/cmp-server/config/plugin.yaml
                  subPath: avp.yaml
                - name: custom-tools
                  mountPath: /usr/local/bin/argocd-vault-plugin
                  subPath: argocd-vault-plugin   
    ```

##### 单独安装argoCD

在安装时需要修改如下helm values:

```yaml
reposerver:                                        # 需要修改repo-server的配置
  volumes:                                         # 添加volumes
    - name: cmp-plugin
      configMap:
        name: cmp-plugin
    - name: custom-tools
      emptyDir: {}
      
  initContainers:
    - name: init-vault-plugin                       # 新增一个initContainer插件二进制文件的拷贝
      image: release.daocloud.io/amamba/argocd-vault-plugin:v1.17.0 # 如果是离线环境，还需要添加离线环境的前缀地址
      command:
        - sh
        - '-c'
      args:
        - cp /usr/local/bin/argocd-vault-plugin /custom-tools
      volumeMounts:
        - name: custom-tools
          mountPath: /custom-tools
          
  envFrom:                                           # 将argocd-vault-plugin需要使用的配置以环境变量的形式进行挂载
    - secretRef:
      name: argocd-vault-plugin-credentials
      
  volumeMounts:                                      # 将相关plugin的目录和配置进行挂载
    - name: plugins
      mountPath: /home/argocd/cmp-server/plugins
    - name: tmp
      mountPath: /tmp
    - name: cmp-plugin
      mountPath: /home/argocd/cmp-server/config/plugin.yaml
      subPath: avp.yaml
    - name: custom-tools
      mountPath: /usr/local/bin/argocd-vault-plugin
      subPath: argocd-vault-plugin
            
  extraContainers:  # 添加一个新的container      
    - name: avp      
      image: quay.io/argoproj/argocd:v2.10.4         # 此处的镜像需要与repo-server的镜像保持一致
      command:
        - /var/run/argocd/argocd-cmp-server
      envFrom:
        - secretRef:
            name: argocd-vault-plugin-credentials
      volumeMounts:
        - name: var-files
          mountPath: /var/run/argocd
        - name: plugins
          mountPath: /home/argocd/cmp-server/plugins
        - name: tmp
          mountPath: /tmp
        - name: cmp-plugin
          mountPath: /home/argocd/cmp-server/config/plugin.yaml
          subPath: avp.yaml
        - name: custom-tools
          mountPath: /usr/local/bin/argocd-vault-plugin
          subPath: argocd-vault-plugin

configs:                                            # 除此之外，还需要修改configmap添加插件配置
  cmp:
    plugins:
      argocd-vault-plugin:
        allowConcurrency: true
        discover:
          find:
            command:
              - sh
              - "-c"
              - "find . -name '*.yaml' | xargs -I {} grep \"<path\\|avp\\.kubernetes\\.io\" {} | grep ."
        generate:
          command:
            - argocd-vault-plugin 
            - generate 
            - ./
        lockRepo: false
      argocd-vault-plugin-helm:
        allowConcurrency: true
        discover:
          find:
            command:
              - sh
              - "-c"
              - "find . -name 'Chart.yaml' && find . -name 'values.yaml'"
        generate:
          command:
            - sh
            - "-c"
            - |
              helm template $argocd_APP_NAME -n $ARGOCD_APP_NAMESPACE ${argocd_ENV_HELM_ARGS} . |
              argocd-vault-plugin generate -
        lockRepo: false  
```

#### 管理员配置敏感信息

在创建 GitOps 应用之前，需要管理员提前设置好敏感数据。如：

使用 vault：

```shell
vault kv put secret/test-secret username="xxxx" password="xxxx" 
```

使用 secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret   
  namespace: default
data:
  password: dGVzdC1wYXNz
  username: dGVzdC1wd2Q=
type: Opaque
```

#### 修改 Git 仓库中的 manifest 文件

使用步骤如下：

修改 Git 仓库中的 manifest 文件，将敏感信息替换为占位符的形式，如果后端存储采用的是 vault，示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret-vault
  annotations:
    avp.kubernetes.io/path: "secret/data/test-secret"    
    avp.kubernetes.io/secret-version: "2"
stringData:  
  password: <password>     
  username: <username>
```

说明：

- 需要添加对应的annotations，用于插件识别，annotation说明：
    - `avp.kubernetes.io/path`：指定敏感信息的路径，如后端存储使用vault，此路径为vault中的路径. 值可以通过`vault kv get secret/test-secret`得到。通过需要在secret后面加上/data.
    - `avp.kubernetes.io/secret-version`：指定敏感信息的版本，如果后端存储支持版本管理，可以指定版本号
- `stringData`中的`password`和`username`为敏感信息的key，`<password>`和`<username>`为占位符， `<password>` <> 中的password为指定vault中的key

如果后端存储使用的是 kubernetes secret，示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret-k8s
  annotations:
    avp.kubernetes.io/path: "default/test-secret"   
stringData:  
  password: <password>     
  username: <username>
```

说明：

- kubernetes secret不支持版本管理，因此`avp.kubernetes.io/secret-version`是无效的
- `avp.kubernetes.io/path`中的值为secret的名称空间和名称, 如`default/test-secret` 表示在default名称空间下的`test-secret`secret。 这个secret通常由管理员创建,并且部署在指定权限的namespace中，不会保存在Git仓库中。
- `stringData` 中的占位符与vault的占位符一致

支持的 annotations 如下：

| Annotation                       | 描述                                    |
|----------------------------------|---------------------------------------|
| avp.kubernetes.io/path           | vault 的path路径                         |
| avp.kubernetes.io/ignore         | 为true则忽略渲染                            |
| avp.kubernetes.io/kv-version     | KV存储引擎的版本                             |
| avp.kubernetes.io/secret-version | 指定value的版本                            |
| avp.kubernetes.io/remove-missing | 对于secret和configmap而言，忽略key未在vault中的报错 |

同时在 <placeholder> 中还支持使用函数，如 `<password | base64>` 表示将 password 的值进行 base64 编码。

#### 查看部署结果

```shell
$ > kubectl get secret test-secret-k8s -o yaml | yq eval ".data" -
> password: dGVzdC1wYXNz
  username: dGVzdC1wd2Q=
```

可以看到secret中的敏感信息已经被替换为实际的值。

#### 敏感信息更新

!!! note

    如果敏感信息发生变更，argocd是**无法感知**到的（即使创建的Application选择自动同步），需要进入argocd的后台页面，点击`hard-refresh`按钮，再点击`sync`按钮才能进行同步。

## 依赖于项目或工具本身的加解密能力

这种方式实现的有很多，最大的优点就是他们不与argocd绑定，但是仍然可以采用GitOps的方式进行部署。本文中采用[sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)来实现。

sealed-secrets 包含两个工具：

- 一个controller, 用户加解密和创建secret
- 一个客户端工具kubeseal

### 安装

1. 安装controller

    ```shell
    helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
    helm install sealed-secrets -n kube-system --set-string fullnameOverride=sealed-secrets-controller sealed-secrets/sealed-secrets
    ```

2. 安装客户端工具

    ```shell
    KUBESEAL_VERSION='0.26.1'
    wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz" \
    tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal \
    sudo install -m 755 kubeseal /usr/local/bin/kubeseal
    ```

### 使用

管理员生成加密的CR文件：

```shell
# 使用命令行工具将secret加密
kubectl create secret generic mysecret -n argo-cd --dry-run=client --from-literal=username=xxxx -o yaml | \
    kubeseal \
      --controller-name=sealed-secrets-controller \  # 注意名称和命名空间
      --controller-namespace=kube-system \
      --format yaml > mysealedsecret.yaml
```

生成的文件如下：

 ```yaml
 apiVersion: bitnami.com/v1alpha1
 kind: SealedSecret
 metadata:
   name: mysecret
   namespace: argo-cd
 spec:
   encryptedData:
     username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq.....  # 通过kubeseal生成的密文
   template:                                          # 除此以外，还可以指定生成secret的模板，类似于pod template
     type: kubernetes.io/dockerconfigjson
     immutable: true
     metadata:
       labels:
         "xxxx":"xxxx"
       annotations:
         "xxxx":"xxxx"  
 ```

加密的数据采用的是 **非对称加密** 的方式，只有 controller 才能解密，因此可以放心的将加密后的数据存放在 Git 仓库中。

当 argocd 同步时，sealed controller 会根据 `SealedSecret` 去生成 secret。
最终生成的 secret 中的数据会被解密，因此当敏感信息发生变更后，仅需要更新 Git 仓库中的 `SealedSecret` 即可实现自动同步。
