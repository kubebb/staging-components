## 日志组件：

* [ingress+rbac(推荐模式)](#test-with-ingress-and-rbac)
* [ingress+searchguard](#test-with-ingress-and-searchguard)
* [ingress-without-auth](#test-with-ingress-without-auth)

### 功能描述
1. helm package包含了安装elasticsearch和对外暴露elasticsearch服务的ingress资源文件;
2. 支持配置的组件包括以下几个： 
  > - rbacSidecar(.Values.rbacSidecar.enabled): 在elasticsearch client节点上安装kube-rbac-proxy sidecar 用来认证用户token
  > - searchGuard(.Values.elasticsearch.secure): 提供elasticsearch安全通信
  > - ingress(.Values.ingress.enabled): 通过ingress暴露elasticsearch服务，按需开启，默认为 true

rbacSidecar 和 elasticsearch.secure 仅能使用其中一种，比如:
> - .Values.rbacSidecar.enabled: true, .Values.elasticsearch.secure: false
> - .Values.rbacSidecar.enabled: false, .Values.elasticsearch.secure: true

### values 字段解释

```yaml
operatorName: elasticsearch-operator # operator名称
namespace: addon-system #operator部署的ns,需要根据具体情况设置 
watchNamespace: addon-system #operator watch的ns(目前应该是cluster scope)，需要根据具体情况设置
replicaCount: 1

kubectlImage: hub.tenxcloud.com/system_containers/kubectl:v1.20.8 #kubectl镜像, 用来执行一些命令

image:
  repository: hub.tenxcloud.com/system_containers/elasticsearch-operator:v5.4.1 #operator镜像地址
  pullPolicy: IfNotPresent

serviceAccount: #不用处理
  create: true
  annotations: {}
  name: "elasticsearch-operator" #不用处理

elasticsearch: #部署elasticsearch的时候的配置
  name: es #资源名称
  namespace: addon-system #部署在哪个namespace
  affinity:  #不用处理
  allInOne: true #如果是单机部署 配置为true 集群配置为false
  clientNodeReplicas: 1 #如果集群模式 es client的个数
  masterNodeReplicas: 1 #如果集群模式 es master的个数
  dataNodeReplicas: 1 #如果集群模式 es data的个数
  dataDiskSize: 50Gi #elasticsearch data pod磁盘大小
  elasticSearchImage: hub.tenxcloud.com/system_containers/elasticsearch:7.10.1-ik #elasticsearch镜像地址
  enabled: true
  secure: true
  javaOptions: -Xms1g -Xmx1g #JVM 堆栈配置，根据需要调整
  masterDiskSize: 1Gi #elasticsearch master pod磁盘大小
  resources:
    limits:
      cpu: "1"
      memory: 2Gi
    requests:
      cpu: "1"
      memory: 2Gi
  storage:
    storageClass: openebs-hostpath # 配置集群内可用的 storageclass

rbacSidecar: #kube-rbac-proxy配置
  enabled: false 
  image: hub.tenxcloud.com/system_containers/kube-rbac-proxy:v0.13.0-32f11472 #kube-rbac-proxy镜像地址
  imagePullPolicy: IfNotPresent
  args: #kube-rbac-proxy启动参数
    - "--secure-listen-address=0.0.0.0:8443"
    - "--upstream=http://127.0.0.1:9200"
    - "--logtostderr=true"
    - "--oidc-issuer=https://oidc.172.22.96.16.nip.io" #这里是oidc-server的地址,需要根据具体环境配置
    - "--oidc-clientID=bff-client" #需要根据具体环境配置,保持不变即可
    - "--oidc-ca-file=/etc/oidc-server/tls/ca.crt" #需要根据具体环境配置,保持不变即可
    - "--token-passthrough"
    - "--v=1"
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

fluentd: #fluentd配置
  enabled: true #是否开启fluentd
  withAuditLog: true #fluend是否收集audit日志
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role.kubernetes.io/agent
                operator: DoesNotExist
  image: hub.tenxcloud.com/system_containers/fluentd-elk:v5.0-kfk
  imagePullPolicy: IfNotPresent
  resources:
    limits:
      cpu: "1"
      memory: 2Gi
    requests:
      cpu: 100m
      memory: 256Mi
  tolerations:
    - effect: NoSchedule
      operator: Exists
    - effect: NoExecute
      operator: Exists

ingress: 
  enabled: true
  className: "" # 1.33版本的ingress,使用className，annotations需要去掉kubernetes.io/ingress.class注解
  ingressName: "ingress-es"
  annotations:
    # 0.44版本的ingress，使用注解的方式，这个注解的值根据ingress-controller里面配置的值来确定
    ingressClass: nginx-ingress-wwixjqtizm #需要根据具体环境配置
  # kubernetes.io/tls-acme: "true"
  labels: {}
  hosts:
    - host: logging.nip.io #这里是elasticsearch通过ingress暴露出来的host,命名规范logging.<ingress_controller_ip>.nip.io,这里ingress_controller_ip需要根据具体的环境获取
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #- secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

```
## <a id=test>安装测试</a>

### <a id=test-with-ingress-and-rbac>1. 使用 ingress 与 RBAC 认证方式</a>
> - .Values.rbacSidecar.enabled: true
> - .Values.elasticsearch.secure: false
> - .Values.ingress.enabled false

#### 依赖镜像
> hub.tenxcloud.com/system_containers/elasticsearch-operator                        v5.4.1              89aeaffad933   2 hours ago     30.8MB
> hub.tenxcloud.com/system_containers/elasticsearch                                 7.10.1-ik           3bf941c09b95   8 months ago    963MB
> hub.tenxcloud.com/system_containers/kubectl                                       v1.20.8             403754878e80   3 months ago    112MB
> hub.tenxcloud.com/system_containers/fluentd-elk                                   v5.0-kfk            63cd90e77b9c   18 months ago    347MB
> hub.tenxcloud.com/system_containers/kube-rbac-proxy                               v0.13.0-32f11472    d00c78c0989a   30 minutes ago   54.2MB

####  [调整values.yml](#value)
> - .Values.rbacSidecar.enabled: true
> - .Values.elasticsearch.secure: false
> - .Values.ingress.enabled true

kube-rbac-proxy的配置在rbacSidercar这一段
```yaml
rbacSidecar:
  enabled: true
  image: hub.tenxcloud.com/system_containers/kube-rbac-proxy:v0.13.0-32f11472
  imagePullPolicy: IfNotPresent
  certSecretName: oidc-server-root-secret #看下面获取参数描述
  args:
    - "--insecure-listen-address=0.0.0.0:8443"
    - "--upstream=http://127.0.0.1:9200"
    - "--oidc-issuer=https://oidc.172.22.96.16.nip.io" #看下面获取参数描述
    - "--oidc-clientID=bff-client"
    - "--oidc-ca-file=/etc/oidc-server/tls/ca.crt"
    - "--token-passthrough"
    - "--logtostderr=true"
    - "--v=1"
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi
```
获取参数 --oidc-issuer，设置为 oidc 服务的地址，一般由 u4a-component 提供

获取参数 certSecretName: 这个secret需要从u4a-system这个namespace中获取, kubectl get secret oidc-server-root-secret -n u4a-system -o yaml > oidc-server-root-secret.yaml, 将yaml里边的namespace信息改为 addon-system，并通过 kubectl apply -f oidc-server-root-secret.yaml 创建到 addon-system 中。

#### 部署 elasticsearch
开始安装:
- 进入 es-logging/helm-chart 目录，执行 helm install es . -n addon-system
- helm list -n addon-system,可以看到release数据

```
# kubectl get pod -n addon-system查看pod状态(这里有一个job依赖es的状态，可能需要一点时间)
# wait es-sgadmin-es job completed
# kubectl get pod -n addon-system
NAME                                        READY   STATUS      RESTARTS   AGE
es-allinone-es-0                            1/1     Running     0          5m40s
es-elasticsearch-operator-88c7cccf6-5scjt   1/1     Running     0          5m51s
es-sgadmin-es-mmnc4                         0/1     Completed   0          28s

```

#### 访问 elasticsearch
1. 获取fluentd sa secret
```
SECRET_NAME=$(kubectl get secret -n addon-system | grep fluentd | awk '{print $1}')
```
2. 获取访问elasticsearch的token
```
TOKEN=$(kubectl get secret $SECRET_NAME -n addon-system -o yaml | grep "token:" | awk -F": " '{print $2}' | base64 -d)
```
3. 获取ingress信息
```
ES_HOST=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $3}')
INGRESS_IP=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $4}')
```
4. 访问elasticsearch
```bash
curl -H "Authorization: Bearer $TOKEN" http://$INGRESS_IP -H "Host: $ES_HOST"
```
会返回如下提示信息，说明日志服务正常启动：
```json
{
	"name": "es-allinone-es-0",
	"cluster_name": "es",
	"cluster_uuid": "ATBDAzVHQeSDb7gaKdgNUw",
	"version": {
		"number": "7.10.1",
		"build_flavor": "default",
		"build_type": "tar",
		"build_hash": "7a15d2a",
		"build_date": "2020-08-12T07:27:20.804867Z",
		"build_snapshot": false,
		"lucene_version": "7.7.3",
		"minimum_wire_compatibility_version": "5.6.0",
		"minimum_index_compatibility_version": "5.0.0"
	},
	"tagline": "You Know, for Search"
}
```

### <a id=test-with-ingress-without-auth> 2. ingress 下不开启 RBAC 认证</a>
> - .Values.rbacSidecar.enabled: false
> - .Values.elasticsearch.secure: false
> - .Values.ingress.enabled true

#### [准备镜像(参考之前的镜像)](#image)
> hub.tenxcloud.com/system_containers/elasticsearch-operator                        v5.4.1              89aeaffad933   2 hours ago     30.8MB
> hub.tenxcloud.com/system_containers/elasticsearch                                 7.10.1-ik           3bf941c09b95   8 months ago    963MB
> hub.tenxcloud.com/system_containers/kubectl                                       v1.20.8             403754878e80   3 months ago    112MB
> hub.tenxcloud.com/system_containers/fluentd-elk                                   v5.0-kfk            63cd90e77b9c   18 months ago    347MB

####  [调整values.yml](#value)
> - .Values.rbacSidecar.enabled: false
> - .Values.elasticsearch.secure: false
> - .Values.ingress.enabled true

#### 访问 elasticsearch
1. 获取ingress信息
``````
ES_HOST=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $3}')
INGRESS_IP=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $4}')
``````
2. 访问elasticsearch
```bash
curl http://$INGRESS_IP -H "Host: $ES_HOST"
```
会返回如下提示信息，说明日志服务正常启动：
```
{
	"name": "es-allinone-es-0",
	"cluster_name": "es",
	"cluster_uuid": "ATBDAzVHQeSDb7gaKdgNUw",
	"version": {
		"number": "7.10.1",
		"build_flavor": "default",
		"build_type": "tar",
		"build_hash": "7a15d2a",
		"build_date": "2020-08-12T07:27:20.804867Z",
		"build_snapshot": false,
		"lucene_version": "7.7.3",
		"minimum_wire_compatibility_version": "5.6.0",
		"minimum_index_compatibility_version": "5.0.0"
	},
	"tagline": "You Know, for Search"
}
```

### <a id=test-with-ingress-and-searchguard>3. 使用 ingress 与 searchguard 认证的模式</a>
> - .Values.rbacSidecar.enabled: false
> - .Values.elasticsearch.secure: true
> - .Values.ingress.enabled true

#### <a id=image>准备镜像</a>
> hub.tenxcloud.com/system_containers/elasticsearch-operator                        v5.4.1              89aeaffad933   2 hours ago     30.8MB
> hub.tenxcloud.com/system_containers/elasticsearch                                 7.10.1-ik           3bf941c09b95   8 months ago    963MB 
> hub.tenxcloud.com/system_containers/kubectl                                       v1.20.8             403754878e80   3 months ago    112MB
> hub.tenxcloud.com/system_containers/fluentd-elk                                   v5.0-kfk            63cd90e77b9c   18 months ago    347MB
#### <a id=value> 调整values.yml文件</a>
解压helm package进入到目录(tar -zxvf elasticsearch-operator-v0830.tgz && cd elasticsearch-operator)修改values.yaml文件 \
<b>关注加注释的部分</b>,调整加注释的那些字段,注释中标注了required表示一定需要修改的

```yaml
operatorName: elasticsearch-operator
namespace: addon-system  #elasticsearch-operator安装的namespace,eg.addon-system
watchNamespace: addon-system #elasticsearch安装的namespace,可以同上面value一样,eg.addon-system
replicaCount: 1

kubectlImage: hub.tenxcloud.com/system_containers/kubectl:v1.20.8 #kubectl镜像, required

image:
  repository: hub.tenxcloud.com/system_containers/elasticsearch-operator:v5.4.1 #elasticsearch-operator镜像地址; required
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  annotations: {}
  name: "elasticsearch-operator"

elasticsearch:
  name: es
  namespace: addon-system #elasticsearch安装的.Values.watchNamespace,可以同value一样,eg.addon-system
  affinity:
  allInOne: true #true: elasticsearch单机模式;false: elasticsearch集群模式; required
  clientNodeReplicas: 1 #elasticsearch集群模式, client节点个数
  masterNodeReplicas: 1 #elasticsearch集群模式, master节点个数
  dataNodeReplicas: 1 #elasticsearch集群模式, data节点个数
  dataDiskSize: 50Gi
  elasticSearchImage: hub.tenxcloud.com./system_containers/elasticsearch:7.10.1-ik #elasticsearch镜像地址; required
  enabled: true
  secure: true
  javaOptions: -Xms1g -Xmx1g
  masterDiskSize: 1Gi
  resources:
    limits:
      cpu: "1"
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 1Gi
  storage:
    storageClass: openebs-hostpath

rbacSidecar:
  enabled: false
  image: hub.tenxcloud.com/system_containers/kube-rbac-proxy:v0.13.0-32f11472
  imagePullPolicy: IfNotPresent
  certSecretName: oidc-server-root-secret
  args:
    - "--insecure-listen-address=0.0.0.0:8443"
    - "--upstream=http://127.0.0.1:9200"
    - "--oidc-issuer=https://oidc.172.22.96.16.nip.io"
    - "--oidc-clientID=bff-client"
    - "--oidc-ca-file=/etc/oidc-server/tls/ca.crt"
    - "--token-passthrough"
    - "--logtostderr=true"
    - "--v=1"
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

fluentd:
  enabled: true # fluetnd默认安装的,如果只是部署elasticsearch这里可以配置为false
  withAuditLog: true
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role.kubernetes.io/agent
                operator: DoesNotExist
  image: hub.tenxcloud.com/system_containers/fluentd-elk:v5.0-kfk
  imagePullPolicy: IfNotPresent
  resources:
    limits:
      cpu: "1"
      memory: 2Gi
    requests:
      cpu: 100m
      memory: 256Mi
  tolerations:
    - effect: NoSchedule
      operator: Exists
    - effect: NoExecute
      operator: Exists

ingress:
  enabled: true
  className: ""
  ingressName: "ingress-es"
  annotations:
    ingressClass: nginx-ingress-wwixjqtizm #k8s集群中暴露服务的ingress-controller deployment名称, required
  labels: {}
  hosts:
    - host: logging.nip.io #这里是elasticsearch通过ingress暴露出来的host,命名规范logging.<ingress_controller_ip>.nip.io,这里ingress_controller_ip需要根据具体的环境获取
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #- secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```

#### 访问elasticsearch
1. 获取elasticsearch认证密码
```
PASSWD=$(kubectl get secret es-certs-es -n addon-system -o yaml | grep password | awk -F ': ' '{print $2}' | base64 -d)
```
2. 获取ingress信息
```
ES_HOST=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $3}')
INGRESS_IP=$(kubectl get ingress ingress-es -n addon-system | grep ingress-es | awk '{print $4}')
```
3. 访问elasticsearch
```bash
curl -u"admin:$PASSWD" http://$INGRESS_IP -H "Host: $ES_HOST"
```
会返回如下提示信息，说明日志服务正常启动：
```json
{
	"name": "es-allinone-es-0",
	"cluster_name": "es",
	"cluster_uuid": "ATBDAzVHQeSDb7gaKdgNUw",
	"version": {
		"number": "7.10.1",
		"build_flavor": "default",
		"build_type": "tar",
		"build_hash": "7a15d2a",
		"build_date": "2020-08-12T07:27:20.804867Z",
		"build_snapshot": false,
		"lucene_version": "7.7.3",
		"minimum_wire_compatibility_version": "5.6.0",
		"minimum_index_compatibility_version": "5.0.0"
	},
	"tagline": "You Know, for Search"
}

```
