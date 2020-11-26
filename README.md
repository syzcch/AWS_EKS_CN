# AWS_EKS_CN
## Install EKS
- We choose Kubernates 1.17 in this demo
- With the help of eksctl, we could set up EKS in a easy way.
```
eksctl create cluster --name <cluster-name> --version 1.17 --region <region> --nodegroup-name perf-test-ng --node-type <node type> --nodes <node number> --nodes-min <min value> --nodes-max <max value> --ssh-public-key <your key pair> --alb-ingress-access --managed --vpc-public-subnets <yoursubnets> --asg-access
```
## Set up alb
- The latest version of AWS alb has a new feature that supporting one alb map many services, we will show it later.
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
    --policy-document file://iam-policy.json
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

## Intall HPA
