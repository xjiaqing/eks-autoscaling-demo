###  压测脚本
```
kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --namespace=prometheus \
  --restart=Never -- -c 10 -q 5 -z 60m http://ac0740daf31904f76b0edf410a3e4b72-173452489.ap-southeast-1.elb.amazonaws.com
```

###  观察压测效果
```
kubectl get hpa ui --watch
```

### 安装 Prometheus 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus -n prometheus -f values.yaml
```

### 访问 Prometheus
```
export POD_NAME=$(kubectl get pods --namespace prometheus -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace prometheus port-forward $POD_NAME 9090
```

### 安装 Grafana
```
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -n prometheus
```

### 获取 Grafana 登录密码
```
kubectl get secret --namespace prometheus grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 访问 Grafana
```
export POD_NAME=$(kubectl get pods --namespace prometheus -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     
kubectl --namespace prometheus port-forward $POD_NAME 3000

```

### Grafana IDs:

Kubernetes Pod Overview: 6781

Node Exporter Full: 1860


### kubeops 
```
git clone https://codeberg.org/hjacobs/kube-ops-view

kubectl apply -k kube-ops-view/deploy

# 需要更新创建好的 clusterrolebinding 的 namespace 
k delete clusterrolebinding kube-ops-view
k create clusterrolebinding kube-ops-view --clusterrole kube-ops-view --serviceaccount prometheus:kube-ops-view
```

### kube-ops-view service
```
echo 'apiVersion: v1
kind: Service
metadata:
  name: kube-ops
  namespace: prometheus
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  loadBalancerClass: service.k8s.aws/nlb
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    application: kube-ops-view
    component: frontend' | kubectl apply -f -
```

### Load Balancer controller

> https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/deploy/installation/#network-configuration

1. 创建 Service Account
```
eksctl create iamserviceaccount \
--cluster=sin-eks-cluster \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::867533378352:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region ap-southeast-1 \
--approve
```

2. 使用 Helm 安装 controller 
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=sin-eks-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```


