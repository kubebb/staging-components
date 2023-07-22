### 简介
该监控部署包用于安装监控系统系统的组件，默认安装 monitoring-operator、victoriametrics,选装 nodeExporter、metricsServer、kube-state-metrics、grafana,集群已有的组件不需要安装;
各个组件的作用如下：
- nodeExporter：采集到主机的运行指标如 CPU、内存、磁盘等信息；
- victoriametrics：一个快速高效、经济并且可扩展的监控解决方案和时序数据库,负责监控数据的抓取、存储、查询，并可以根据告警规则触发告警；
- metricsServer： Kubernetes 集群核心监控数据的聚合器，定时从 Kubelet 的 Summary API 采集指标信息），可以通过 Metrics API 的形式获取 Metrics 数据；
- kube-state-metrics：采集 deployment，Pod、daemonset、cronjob 等 k8s 资源对象的监控数据，提供监控指标；
- grafana:一个可视化工具，它提供了强大和优雅的方式去创建、共享、浏览数据，并提供了很多漂亮的模板，当需要直接查看监控数据时候，可以装上；
- monitoring-operator: 负责管理上述监控组件.

### 安装步骤

#### 前置条件
- 如果监控组件 vmselect 如果开启了 sidecar，kube-rbac-proxy 支持 OIDC，则需要提前部署好 OIDC 相关的内容，可以通过执行
  ```
  kubectl  get pod -n u4a-system
  ```
  查看是否有 oidc-server，检查相关组件是否已经安装好；
  
- 租户组件部署好后，需要将 capsule pod 重建下，否则创建 namespace system-monitoring 时可能会报错；
  
- 如果需要使用 ingress，则需要提前部署好 ingress-controller；

- vmstorage 需要进行数据持久化，需要提前准备好 StorageClass；

- 创建好 Group observability，该组具有访问监控数据的权限；



#### 1.准备镜像,push 到对应环境的 harbor 仓库
- 需要以下镜像
```
172.22.50.227/system_containers/vm-operator:v0.14.2-tdsfa
172.22.50.227/system_containers/monitoring-operator:v5.6.0-tdsfa
172.22.50.227/system_containers/prom-rule-reloader:v0.1.2
172.22.50.227/system_containers/kube-rbac-proxy:v0.13.0-32f11472
172.22.50.223/release-5.3.x/node-exporter:v0.17.0 (选装了才需要)
172.22.50.223/release-5.4/kube-state-metrics:v1.9.7 (选装了才需要)
172.22.50.223/release-5.3.x/alertmanager:v0.20.0
172.22.50.223/release-5.4/metrics-server:v0.4.1 (选装了才需要)
172.22.50.223/release-5.3.x/grafana:6.4.3(选装了才需要)
172.22.50.223/release-5.3.x/configmap-reload:v0.3.0
172.22.50.223/release-5.3.x/prometheus-config-reloader:v0.42.0
172.22.50.223/release-5.3.x/vm-operator:v0.14.2
172.22.50.223/release-5.3.x/vminsert:v1.58.0-cluster
172.22.50.223/release-5.3.x/vmstorage:v1.58.0-cluster
172.22.50.223/release-5.3.x/vmselect:v1.58.0-cluster
172.22.50.223/release-5.3.x/vmagent:v1.58.0
172.22.50.223/release-5.3.x/vmalert:v1.58.0
172.22.50.223/release-5.3.x/vmui:v0.1.0
```

#### 2.获取 helm 包，并解压
```
tar zxvf monitoring-operator-0.1.0.tgz
cd monitoring-operator
```

#### 3.修改 charts 包的 values.yaml
参照 values.yaml 里面的注释，主要有以下内容需要修改：

- 根据实际环境，修改镜像地址；
- 带有 enabled 的是可以控制改组件是否可以启用，false 则不安装，true 会安装，没有 enabled 参数会默认装上；
- 如果开启 nodePort，先检查端口是否被占用，不使用设置为 0 即可；
- 如果开启 ingress，需要修改 ingress 资源的注解，注解 key 是 kubernetes.io/ingress.class。注解的值可以查看 ingress-controller 的 deploy 里面的 args 参数，如
  ```
  kubeclt  edit  deploy -n kube-system ingress-urygcdmyts
  ```
  取 args 里面的值- --ingress-class=nginx-ingress-urygcdmyts，nginx-ingress-urygcdmyts 就是要填入注解的值； 
  
#### 4.创建 namesapce
```  
kubectl --as=admin  --as-group=iam.tenxcloud.com create -f - <<EOF 
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: system-monitoring
EOF 
```
- 如果创建 ns 前就存在，可以之前部署过监控，为了确保后续不会报错，先清除旧的 system-monitoring 下的资源，并删除 vm 相关的 crd，查找 vm 的 crd 命令是
``
  kubectl  get  crd | grep victoriametrics.com
``

#### 5.生成 ca 证书(只有 vmselect 开启了 sidecar，支持 oidc 参数时需要)
kube-rbac-proxy 支持 OIDC，args 需要设置参数 oidc-issuer、oidc-clientID、oidc-ca-file，若 oidc-server 部署在 u4a-system 下，可以这样去获取相关的参数，供参考：

- 生成证书：
```
kubectl get secret -n u4a-system  oidc-server-root-secret  -oyaml > oidc-sidecar-secret.yaml

修改 yaml 的 namesapce 为 system-monitoring，创建一个新的 secret

kubectl create -f oidc-sidecar-secret.yaml
```
- oidcIssuer,oidcClientID 参数的获取
```
kubectl  get cm -n u4a-system   oidc-server -o yaml
```
oidcIssuer 取其中的 issuer 的内容即可，比如：https://oidc.192.168.90.217.nip.io

oidcClientID 取其中的 staticClients 下的 id 内容即可，比如 bff-client

#### 6.helm install
- 执行 helm 命令，monitoring-operator 是应用的名称，根据实际需要修改
```
helm install monitoring-operator -n system-monitoring ./
```

#### 7.检查组件是否运行成功
```
kubectl get po -n system-monitoring
```
检查的 Pod 是否正常运行；

#### 8.功能验证
- 部署成功后，可以通过 ingress 地址去方式访问数据,查看 ingress 的 hosts 地址命令如下：

  ```
  kubectl  -n system-monitoring get ingress
  ```
   如果 vmselect 开启了 nodePort，那么也可以通过主机 IP：nodePort 的方式去访问监控数据

- 将用户加入组 observability，该组具有访问监控数据的权限，获取用户 token，访问监控数据带上 token，验证权限，没有权限则出现 Unauthorized;
  请求命令参考：
  ```
  curl -k "monitoring.192.168.90.217.nip.io/select/0/prometheus/api/v1/query" -d "query=up" -H"Authorization: bearer eyJhbGciOi..."
  ```

