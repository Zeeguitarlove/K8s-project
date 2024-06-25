######                ###### 
###### PRE-REQUISITES ######

kubectl – A command line tool for working with Kubernetes clusters.

eksctl – A command line tool for working with EKS clusters that automates many individual tasks.

AWS CLI – A command line tool for working with AWS services, including Amazon EKS.


###### Creating an EKS Cluster with Fargate ###### 

eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --zones "us-east-1a,us-east-1b,us-east-1c,us-east-1d,us-east-1f" \
  --fargate

###### Updating Kubeconfig for EKS Cluster ######

aws eks update-kubeconfig --name demo-cluster --region us-east-1

###### Creating a Fargate Profile for the Cluster ######

eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048

###### Deploying the 2048 Sample Application ######

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

###### Associating IAM OIDC Provider with the Cluster ######

eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve

we need to have IAM OIDC provider because ALB controller that running need to talk to AWS resource.
For it to talk it need to have IAM integrated
Therefore we integrate IAM OIDC


###### Downloading IAM Policy for AWS Load Balancer Controller ######

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

###### Creating IAM Policy for AWS Load Balancer Controller ######

aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json

###### Creating IAM Service Account for AWS Load Balancer Controller ######

eksctl create iamserviceaccount \
--cluster=demo-cluster \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve

###### Adding Helm Repository for EKS Charts ######

helm repo add eks https://aws.github.io/eks-charts

###### Updating Helm Repositories ######

helm repo update eks

###### Installing AWS Load Balancer Controller via Helm ######

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0c77dfe245a03a9a3

###### Check the Deployments ######
kubectl get deployment -n kube-system aws-load-balancer-controller

###### Check the Pods ######

kubectl get pods -n kube-system

###### Check the Address assoaciated to Ingress ######

kubectl get ingress -n game-2048

###### Fetch the Address on which the App is running and paste it on your browser ######

Example:- http://k8s-game2048-ingress2-bcac0b5b37-1116519834.us-east-1.elb.amazonaws.com/

###### Why did I use Fargate over HPA ###### 

Using AWS Fargate over Horizontal Pod Autoscaling (HPA) is advantageous. 
Here are several reasons why I chose AWS Fargate instead of HPA:-

1. Serverless Computing: AWS Fargate is a service that lets you run containers without managing the servers they run on. It takes care of handling the infrastructure and scaling, so you can focus on deploying and running your containers. You don't need to manually adjust how much computing power is allocated to your containers, which is something you'd normally have to do with HPA.

2. Cost Efficiency: Fargate charges you based on how much CPU and memory your containers use. It automatically adjusts resources to match your workload, which helps you save money by not paying for more resources than you actually need.

3. Simplicity: Using Fargate makes managing Kubernetes easier because you don't have to set up or maintain EC2 instances (virtual servers). AWS handles all the infrastructure work like scaling and fixing problems behind the scenes.

4. Automatic Scaling: Fargate can increase or decrease the number of containers running your tasks based on how much CPU and memory they're using. You can set rules so it adjusts automatically as demand changes. This is similar to HPA, but it works at the level of individual tasks (containers) rather than entire groups of containers (pods in Kubernetes).

5. Isolation and Security: Each task you run with Fargate operates in its own separate environment. This setup provides better security and isolation compared to traditional Kubernetes pods, which share resources on the same server nodes.

###### ######

###### ######

###### ######

###### ######