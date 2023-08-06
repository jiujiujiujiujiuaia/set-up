## 1.什么是KEDA

KEDA：Kubernetes-based Event Driven Autoscaler.

KEDA是一种基于事件驱动对 K8S 资源对象扩缩容的组件。非常轻量、简单、功能强大，不仅支持基于 CPU / MEM 资源和基于 Cron 定时 HPA 方式，

同时也支持各种事件驱动型 HPA，比如 MQ 、Kafka 等消息队列长度事件，Redis 、URL Metric 、Promtheus 数值阀值事件等等事件源(Scalers)

![img_3.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_3.png)

## 2.为什么需要KEDA

k8s 官方早就推出了 HPA，为什么我们还需要 KEDA 呢？

HPA 在经过三个大版本的演进后当前支持了Resource、Object、External、Pods 等四种类型的指标，演进历程如下：

* autoscaling/v1：只支持基于CPU指标的缩放
* autoscaling/v2beta1：支持Resource Metrics（资源指标，如pod的CPU）和Custom Metrics（自定义指标）的缩放
* autoscaling/v2beta2：支持Resource Metrics（资源指标，如pod的CPU）和Custom Metrics（自定义指标）和 ExternalMetrics（额外指标）的缩放。

如果需要基于其他地方如 Prometheus、Kafka、云供应商或其他事件上的指标进行伸缩，那么可以通过 v2beta2 版本提供的 external metrics 来实现，具体如下：

* 1）通过 Prometheus-adaptor 将从 prometheus 中拿到的指标转换为 HPA 能够识别的格式，以此来实现基于 prometheus 指标的弹性伸缩
* 2）然后应用需要实现 metrics 接口或者对应的 exporter 来将指标暴露给 Prometheus 

可以看到， HPA v2beta2 版本就可以实现基于外部指标弹性伸缩，只是实现上比较麻烦，KEDA 的出现主要是为了解决 HPA 无法基于灵活的事件源进行伸缩的这个问题。

因此，KEDA通过提供一些列开箱即用的scaler，增加了HPA的能力，使得用户可以很方便的根据自定义指标进行扩缩容。

## 3.部署

### 权限鉴权

### (1).创建一个AKS cluster

```shell
az login

az group create --name myResourceGroup --location eastus

az aks create -g myResourceGroup -n myAKSCluster --enable-managed-identity --node-count 3 --enable-addons monitoring --generate-ssh-keys

az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

### (2).部署MIC(Managed Identity Controller)

```shell
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/v1.8.17/deploy/infra/deployment-rbac.yaml

kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/v1.8.17/deploy/infra/deployment.yaml

kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/v1.8.17/deploy/infra/mic-exception.yaml
```

### (3).部署KEDA

```shell
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.9.3/keda-2.9.3.yaml
```

### (4).创建一个LA(Log Analytics)

LA将会是KEDA的external metric source，作为信号来源。KEDA Metric server会周期性的去读取LA获取最新的metric来决定是否要
扩容或者缩容。

```shell
az monitor log-analytics workspace create -g MyResourceGroup -n MyWorkspace
```

### (5).授予AKS默认的Managed Identity访问LA的权限

```shell
## get the managed identity id
$ManagedIdentityId = az aks show -g myResourceGroup -n myAKSCluster --query identityProfile.kubeletidentity.clientId -otsv

$LASubscriptionId = "7f30cbcf0a27-7b30-4391-8312-18ec658f" ## required fill yours
$LAResourceGroupName = "MyResourceGroup" ## required fill yours

az role assignment create --role "Reader" --assignee "${ManagedIdentityId}" --scope "/subscriptions/${LASubscriptionId}/resourcegroups/${LAResourceGroupName}" 
```

### (6).部署Identity K8S yaml

$IdentityResourceId=az identity show -g ${RESOURCE_GROUP} -n ${IDENTITY_NAME} --query id -otsv

```yaml
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: myidentity
spec:
  type: 0
  resourceID: {$IdentityResourceId}
  clientID: ${$ManagedIdentityId}
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: myidentity-binding
spec:
  azureIdentity: myidentity
  selector: myidentity
```
kubectl apply -f xx.yaml

### (7).授予KEDA pod权限访问LA

```shell
kubectl patch deployment keda-operator -n keda --type json -p='[{"op": "add", "path": "/spec/template/metadata/labels/aadpodidbinding", "value": "'${IDENTITY_NAME}'"}]'

kubectl patch deployment keda-metrics-apiserver -n keda --type json -p='[{"op": "add", "path": "/spec/template/metadata/labels/aadpodidbinding", "value": "'${IDENTITY_NAME}'"}]'
```
## 4.测试

### (1).发送Metric到LA

![img.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img.png)

![img_1.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_1.png)

![img_2.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_2.png)

大概等一个五分钟左右

![img_5.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_5.png)

### (2).部署demo deployment和scaled object(CRD)

kubectl apply -f deployment.yaml

![img_6.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_6.png)

kubectl apply -f scaledobject.yaml

![img_9.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_9.png)

![img_10.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_10.png)

### (3).获取metric
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"

![img_7.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_7.png)

kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/example/s0-azure-log-analytics-10c097e6-9e38-4c0a-aaf8-60e8e57d7047?labelSelector=scaledobject.keda.sh%2Fname%3Dkedaloganalytics-scaled-object"

![img_8.png](https://raw.githubusercontent.com/jiujiujiujiujiuaia/jiujiujiujiujiuaia.github.io/master/_posts/pic/KEDA/img_8.png)

## Reference
1.https://www.lixueduan.com/posts/kubernetes/18-keda/