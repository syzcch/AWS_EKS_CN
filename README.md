# AWS_EKS_CN
## Install EKS
- We choose Kubernates 1.17 in this demo
- With the help of eksctl, we could set up EKS in a easy way.
```
eksctl create cluster --name <cluster-name> --version 1.17 --region <region> --nodegroup-name perf-test-ng --node-type <node type> --nodes <node number> --nodes-min <min value> --nodes-max <max value> --ssh-public-key <your key pair> --alb-ingress-access --managed --vpc-public-subnets <yoursubnets> --asg-access
```
## Set up alb
- The latest version of AWS alb has a new feature that supporting one alb map many services, we will show it later.
- We need internal alb, so we need add tag to our subnets like this
```
kubernetes.io/role/internal-elb   1
```
- If you want to create a public one, set it like this
```
kubernetes.io/role/elb   1
```
###
### Repo download
```
Download the repo, cd to right path.
```
### Set up OIDC
```
eksctl utils associate-iam-oidc-provider \
    --region <region> \
    --cluster <cluster-name> \
    --approve
```
### Create policy for alb
```
aws iam create-policy \
    --policy-name <policy name> \
    --policy-document file://iam-policy-cn.json
```
- Note the arn that the command returned and will use in next step
### Create service account
```
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=<service account name> \
  --attach-policy-arn=<the arn> \
  --override-existing-serviceaccounts \
  --approve
```
### Set up cert manager
```
ensure that the image source is  048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn
kubectl apply -f cert-manager.yaml
```
- check the status of it
```
kubectl get pods -ncert-manager
[ec2-user@ip-10-64-144-5 lb-controller]$ kubectl get pods -ncert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-574654dfb4-9kbf7              1/1     Running   0          3d1h
cert-manager-cainjector-7b85ddd65f-b4fdn   1/1     Running   0          3d1h
cert-manager-webhook-865bb8554-5vjxj       1/1     Running   0          3d1h
```
### Install alb-controller
```
Open the file lb-controller-v2.yaml then using your cluster name to replace the current one.
kubectl apply -f lb-controller-v2.yaml
```
- check the result of it
```
kubectl get pods -nkube-system
[ec2-user@ip-10-64-144-5 lb-controller]$ kubectl get pods -nkube-system
NAME                                            READY   STATUS              RESTARTS   AGE
aws-load-balancer-controller-7f895dfd97-fm8jb   1/1     Running             0          3d1h
```
### Install test demo 2048
```
Open the file 2048_full.yaml
Add alb group
alb.ingress.kubernetes.io/group.name: <group name>

kubectl apply -f 2048_full.yaml
```
### Check result
```
[ec2-user@ip-10-64-144-5 lb-controller]$ kubectl get ing -ngame-2048
NAME           HOSTS   ADDRESS                                                                                    PORTS   AGE
ingress-2048   *       internal-k8s-game2048-ingress2-92554a9080-2098904851.cn-northwest-1.elb.amazonaws.com.cn   80      3d1h
```
- Check logs of it
```
kubectl logs -n kube-system   deployment.apps/aws-load-balancer-controller
```
### Clear env(if you need it)
```
kubectl delete -f 2048_full.yaml
```

## Install Monitoring
### Install Helm
```
export PATH=/usr/local/bin:$PATH
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

helm repo list
helm repo update
helm repo add stable https://burdenbear.github.io/kube-charts-mirror/
```
### Install Prometheus
```
kubectl create namespace prometheus

helm install prometheus stable/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```
### Install Grafana
```
kubectl create namespace grafana

helm install grafana stable/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set adminPassword='EKS!sAWSome' \
    --set datasources."datasources\.yaml".apiVersion=1 \
    --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
    --set datasources."datasources\.yaml".datasources[0].type=prometheus \
    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources."datasources\.yaml".datasources[0].access=proxy \
    --set datasources."datasources\.yaml".datasources[0].isDefault=true \
    --set service.type=LoadBalancer 
```
- Then check the result
```
kubectl get service -n grafana

[ec2-user@ip-10-64-144-5 ~]$ kubectl get service -n grafana
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                                                                       PORT(S)        AGE
grafana   LoadBalancer   172.20.38.143   a88ebb6e72c59408981150776a39c45d-1889022542.cn-northwest-1.elb.amazonaws.com.cn   80:30880/TCP   8d
```
- Log into the portal
```
Copy the EXTERNAL-IP and paste it on browser.
Email or username = admin
Password = EKS!sAWSome
```
### Set up a panel
```
左侧面板点击' + '，选择' Import '
Grafana.com Dashboard下输6417，点击“Load”
输入Kubernetes Pods Monitoring作为Dashboard名称
点击change，设置uid
prometheus data source下拉框中选择prometheus
点击' Import '
```
### Clean env if you need it
```
helm uninstall grafana --namespace grafana
helm uninstall prometheus --namespace prometheus
kubectl delete ns grafana
kubectl delete ns prometheus
```

## Intall HPA
### Install metric-server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
kubectl get deployment metrics-server -n kube-system

[ec2-user@ip-10-64-144-5 ~]$ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           7d22h
```
### Set up limit for pod
```
kubectl set resources deployment deployment-2048 --limits=cpu=50m,memory=64Mi
```
### Set up auto-scaling policy
```
kubectl autoscale deployment deployment-2048 --cpu-percent=20 --min=1 --max=6
```
### Monitor it
```
kubectl top pod
kubectl get hpa --watch
```
### Run test case
- Check the number of pod for game-2048
```
[ec2-user@ip-10-64-144-5 ~]$ kubectl get pods -ngame-2048
NAME                               READY   STATUS    RESTARTS   AGE
deployment-2048-79c4dc468c-8zmkl   1/1     Running   0          8d
```
- Run busy box
```
kubectl run -it --rm load-generator --image=busybox /bin/sh --generator=run-pod/v1

while true; do wget -q -O- http://internal-k8s-game2048-ingress2-92554a9080-2098904851.cn-northwest-1.elb.amazonaws.com.cn/; done
```
- Check the hpa
```
[ec2-user@ip-10-64-144-5 ~]$ kubectl get hpa -ngame-2048
NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
deployment-2048   Deployment/deployment-2048   38%/10%   1         6         4          8d
```
- Check the number of pods again
```
[ec2-user@ip-10-64-144-5 ~]$ kubectl get pods -ngame-2048
NAME                               READY   STATUS    RESTARTS   AGE
deployment-2048-79c4dc468c-8zmkl   1/1     Running   0          8d
deployment-2048-79c4dc468c-kw7hr   1/1     Running   0          27s
deployment-2048-79c4dc468c-mdhpd   1/1     Running   0          27s
deployment-2048-79c4dc468c-qqnqb   1/1     Running   0          27s
```
- Stop the busy test
- Check the number again
```
[ec2-user@ip-10-64-144-5 ~]$ kubectl get pods -ngame-2048
NAME                               READY   STATUS    RESTARTS   AGE
deployment-2048-79c4dc468c-8zmkl   1/1     Running   0          8d
```
### Clean env if you need it
```
kubectl delete deployment.apps/deployment-2048 service/deployment-2048 horizontalpodautoscaler.autoscaling/deployment-2048
```

## Auto-scaler
### Install auto-scaler
```
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
### Prepare image of ac
```
sudo docker pull k8s.gcr.io/autoscaling/cluster-autoscaler:v1.17.4

Add tag then push it onto ECR or Dockerhub

704567433200.dkr.ecr.cn-northwest-1.amazonaws.com.cn/management:cluster-autoscaler-v1.17.4
rogerhc/aws-resources:cluster-autoscaler-v1.17.4
```
### Update cluster-autoscaler-autodiscover.yaml
```
 containers:
              #- image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.17.4
              #- image: 704567433200.dkr.ecr.cn-northwest-1.amazonaws.com.cn/management:cluster-autoscaler-v1.17.4
        - image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/autoscaling/cluster-autoscaler:v1.17.3
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/{cluster-name}
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
            - --aws-use-static-instance-list=true
```
### Apply for it
```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```
### Try to use it
- Create a test service
```
cat <<EoF> nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF


kubectl apply -f nginx.yaml
```
- Set replicas for it
```
kubectl get node
NAME                                               STATUS   ROLES    AGE   VERSION
ip-10-64-149-205.cn-northwest-1.compute.internal   Ready    <none>   10d   v1.17.12-eks-7684af
ip-10-64-150-172.cn-northwest-1.compute.internal   Ready    <none>   14m   v1.17.12-eks-7684af

kubectl scale --replicas=10 deployment/nginx-to-scaleout
```
- Check logs 
```
kubectl get pods -l app=nginx -o wide --watch

kubectl -n kube-system logs -f deployment/cluster-autoscaler
```
- Check nodes
```
Waiting for a moment
kubectl get node
NAME                                               STATUS   ROLES    AGE     VERSION
ip-10-64-149-205.cn-northwest-1.compute.internal   Ready    <none>   10d     v1.17.12-eks-7684af
ip-10-64-150-172.cn-northwest-1.compute.internal   Ready    <none>   4m10s   v1.17.12-eks-7684af
ip-10-64-150-87.cn-northwest-1.compute.internal    Ready    <none>   10d     v1.17.12-eks-7684af
```
- Scale down test
```
kubectl scale --replicas=1 deployment/nginx-to-scaleout

waiting for more than 10 mins

kubectl get node
NAME                                               STATUS   ROLES    AGE   VERSION
ip-10-64-149-205.cn-northwest-1.compute.internal   Ready    <none>   10d   v1.17.12-eks-7684af
ip-10-64-150-172.cn-northwest-1.compute.internal   Ready    <none>   14m   v1.17.12-eks-7684af
```
### Clean env if you need it
```
kubectl delete -f nginx.yaml
kubectl delete -f cluster-autoscaler-autodiscover.yaml
```
## Useful links
```
https://www.eksworkshop.com/beginner/080_scaling/test_ca/
https://github.com/wangyu7988/I-Day/tree/master/Devops/EKSworkshop
https://github.com/jansony1/cn-container-lab # ingress
https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html
https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html
https://github.com/aws-samples/eks-workshop-greater-china/blob/master/china/2020_EKS_Launch_Workshop/%E6%AD%A5%E9%AA%A48-%E5%AF%B9%E5%BA%94%E7%94%A8Pod%E5%92%8C%E9%9B%86%E7%BE%A4%E8%BF%9B%E8%A1%8C%E8%87%AA%E5%8A%A8%E6%89%A9%E5%B1%95.md
```
