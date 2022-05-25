# EKS Commands

A collection of useful EKS and related (kubectl, aws eks, eksctl) commands:

## Install typical tools:
kubectl on Amazon Linux 2, amd64:
```
sudo curl --location -o /usr/local/bin/kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
kubectl version --short --client

```
eksctl on Amazon Linux 2, amd64:
```
curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version

```

## Cluster creation with eksctl on an EC2 instance:
```
export AWS_REGION=$(curl --silent http://169.254.169.254/latest/meta-data/placement/region) && echo $AWS_REGION
eksctl create cluster \
--name dev-cluster \
--nodegroup-name dev-nodes \
--node-type t3.small \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--managed \
--version 1.21 \
--region ${AWS_REGION}
```
Create a serverless cluster using Fargate:
see this 12 min. video as a starter: https://www.youtube.com/watch?v=DLmKMBZ_m3w
```
eksctl create-cluster --name eks-fargate-cluster --region us-east-1 --fargate
```

## Cluster deletion with ekstcl:
```
eksctl delete cluster --name=<cluster-name>
```


## Getting an overview over the cluster setup:

List all K8s resources in the **default** namespace (-o wide = give more details):
```
kubectl get all [-o widekube]
```

Same for "web" namespace (--all-namespaces for **all namespaces**):
```
kubectl get all -n web
```

List daemonsets for namespace kube-system:
```
kubectl get daemonsets -n kube-system
```
Dump relevant configs to new directories:
```
for n in $(kubectl get -o=name pvc,configmap,serviceaccount,secret,ingress,service,deployment,statefulset,hpa,job,cronjob)
do
    echo checking $n
    mkdir -p $(dirname $n)
    kubectl get $n -o yaml > $n.yaml
done
```

## Create a kubeconfig backup (default path):
```
cp ~/.kube/config ~/.kube/config.back
```

## Deployment related:
K8s doc: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
Make a simple nginx web server deployment in namespace "web":
```
kubectl create deployment nginx --image=nginx -n web
```
For a quick test, create another nginx deployment using a different name:
```
kubectl create deployment nginx_2 --image=nginx -n web
```
Create a deployment based on a yaml file:
```
kubectl apply -f scripts/nginx-deployment.yaml
```
List all deployments in all namespaces, both as a list and in yaml output:
```
kubectl get deployments --all-namespaces
kubectl get deployments --all-namespaces -o yaml > scripts/all-deployments-new.yaml
# note: deployments are listed in the yaml, one deployment per entry in "items:" list
```
Get details of all deployments (or provide a specific deployment name):
```
kubectl describe deployments
```
**Delete** a deployment
```
kubectl delete deployment <deployment-name>
```

## Node and Pod related
Get nodes (irrespective of EC2 or fargate based nodes):
```
kubectl get nodes
```
Get Pods in all namespaces, (-w = keep watching their state, also works with other commands):
```
kubectl get pod --all-namespaces -w
```

## Make deployment accessible from the outside (expose it):
Expose the above nginx server on port 80 by using an Elastic Load Balancer:
```
kubectl expose deployment nginx --port=80 --name nginx --type=LoadBalancer -n web
```

Get Load Balancer endpoint information for a specific service and namespace:
```
kubectl get service nginx -n web
```

## OIDC (OpenId Connect) related:
List all OIDC providers:
```
aws iam list-open-id-connect-providers
```

Create a new iam role and a corresponding K8s service account 
```
CLUSTER=$(aws eks list-clusters | jq -r .clusters[0]) && echo $CLUSTER
AWS_REGION=$(curl --silent http://169.254.169.254/latest/meta-data/placement/region) && echo $AWS_REGION
eksctl create iamserviceaccount --name aws-s3-read --namespace default --cluster ${CLUSTER} --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --approve --region ${AWS_REGION}
```

## Permissions K8s <-> IAM
Display details of the aws-auth configMap:
```
kubectl describe configmap -n kube-system aws-auth
```

Create namespace web, a "web-admins-role" granting access, and a "web-admins-binding" that grants permissions defined in the role to "web-admins-group"
```
kubectl apply -f ~/scripts/task3/namespace-role-rolebinding.yaml
```

Add a new IAM role to RBAC group mapping to the aws-auth ConfigMap:
```
CLUSTER=$(aws eks list-clusters | jq -r .clusters[0]) && echo $CLUSTER
AWS_REGION=$(curl --silent http://169.254.169.254/latest/meta-data/placement/region) && echo $AWS_REGION
export WEB_ADMIN_ARN=$(aws iam list-roles | jq -r '.[] | .[] | .Arn' | grep -i web) && echo "The ARN for the WebAdminRole is" $WEB_ADMIN_ARN
eksctl create iamidentitymapping --cluster ${CLUSTER} --group web-admins-group --username web-admin --region ${AWS_REGION} --arn ${WEB_ADMIN_ARN}
```
Check the new group in the aws-auth ConfigMap:
```
kubectl describe configmap -n kube-system aws-auth
```

## Create a temp container:
Create a temp container, specify a serviceAccount and open a bash shell into the container. After bash is terminated, remover everything (--rm):
```
kubectl run my-shell --rm -i --tty --image amazonlinux --overrides='{ "spec": { "serviceAccount": "aws-s3-read" } }' -- bash
```

If needed, perform some installations inside of the container:
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.6.2.zip" -o "awscliv2.zip"
yum install unzip less groff -y
unzip awscliv2.zip
./aws/install
```

## Identities:
Verify my current identity:
```
aws sts get-caller-identity
```

## Use Calico, a network policy engine for K8s:
Verify the Calico daemonset is running:
```
kubectl get daemonset calico-node --namespace kube-system
```

## Deny/allow activities:
Deny all traffic for a given namespace (see yaml file in scripts/)
```
kubectl apply -f ~/scripts/task5/deny-traffic.yaml
```

Allow external web access to an nginx pod in a given namespace (see yaml file in scripts/)
```
kubectl apply -f ~/scripts/allow-external-web-access.yaml
```
