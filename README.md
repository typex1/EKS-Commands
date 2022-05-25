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

## Create a kubeconfig backup:
```
cp ~/.kube/config ~/.kube/config.back
```

## Create a deployment:
Make a simple nginx web server deployment in namespace "web":
```
kubectl create deployment nginx --image=nginx -n web
```

## Make deployment accessible from the outside (expose it):
Expose the above nginx server on port 80 by using an Elastic Load Balancer:
```
kubectl expose deployment nginx --port=80 --name nginx --type=LoadBalancer -n web
```

Get Load Balancer endpoint information:
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
