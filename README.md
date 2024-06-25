######                ###### 
###### PRE-REQUISITES ######

kubectl – A command line tool for working with Kubernetes clusters.

eksctl – A command line tool for working with EKS clusters that automates many individual tasks.

AWS CLI – A command line tool for working with AWS services, including Amazon EKS.


###### Creating an EKS Cluster with Fargate ###### 

eksctl create cluster \
  --name "your-cluster-name" \
  --region us-east-1 \
  --zones "us-east-1a,us-east-1b,us-east-1c,us-east-1d,us-east-1f" \
  --fargate

![1 Cluster created](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/8219c6d7-99b9-4a80-a023-f412ff4d6ca8)


###### Updating Kubeconfig for EKS Cluster ######

aws eks update-kubeconfig --name "your-cluster-name" --region us-east-1

![2 Update KubeConfig](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/ab8a8283-e66d-4b4c-9231-821ea9639fde)

###### Creating a Fargate Profile for the Cluster ######

eksctl create fargateprofile \
    --cluster "your-cluster-name" \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048

![3 Fargate profile created](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/fa4e11aa-4935-445e-921e-9f2b513705ce)


###### Deploying the 2048 Sample Application ######

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

![4 Create Ns,deploy,sv,ingress](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/11e826ac-6597-4845-b379-aeb64418ca4a)


###### Associating IAM OIDC Provider with the Cluster ######

eksctl utils associate-iam-oidc-provider --cluster "your-cluster-name" --approve

we need to have IAM OIDC provider because ALB controller that running need to talk to AWS resource.
For it to talk it need to have IAM integrated
Therefore we integrate IAM OIDC

![5 IAM OIDC has been created to access ALB](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/7d513a4a-33a7-4aa2-8d70-2623eaf38a26)


###### Downloading IAM Policy for AWS Load Balancer Controller ######

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

###### Creating IAM Policy for AWS Load Balancer Controller ######

aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json

![6  IAM Policy Created](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/b43a5003-e875-4e10-90e2-08108f4917d3)



###### Creating IAM Service Account for AWS Load Balancer Controller ######

eksctl create iamserviceaccount \
--cluster="your-cluster-name" \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=arn:aws:iam::"your-account-id":policy/AWSLoadBalancerControllerIAMPolicy \
--approve

![7 Created IAM Role and Attached policy to it](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/c3c0c261-c883-47a3-a7c5-c6ef95e7135e)


###### Adding Helm Repository for EKS Charts ######

helm repo add eks https://aws.github.io/eks-charts\

![8 Add Helm Repo](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/9ff8d1fd-cf1e-42f0-b69a-2b1e1c744a07)


###### Updating Helm Repositories ######

helm repo update eks

![9  Update the Helm repo](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/10403cf7-798a-4e55-b49b-a46baae7d5a5)


###### Installing AWS Load Balancer Controller via Helm ######

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName="your-cluster-name" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId="your-VPC-ID"

###### Check the Deployments ######
kubectl get deployment -n kube-system aws-load-balancer-controller

![10  Check the Deployments](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/d6d99629-553e-435b-b6c0-140bc4ae6360)


###### Check the Pods ######

kubectl get pods -n kube-system

![11  Check the Pods](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/30e62d1c-d2c7-4e96-9a24-a5fbfd32ecf1)


###### Check the Address assoaciated to Ingress ######

kubectl get ingress -n game-2048

![12  Check the Address assoaciated to Ingress](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/9ca429b4-3ae7-4aae-97d9-3f7a0fcdb4b5)


###### Fetch the Address on which the App is running and paste it on your browser ######

Example:- http://k8s-game2048-ingress2-bcac0b5b37-1116519834.us-east-1.elb.amazonaws.com/

![13  Browser Link](https://github.com/Zeeguitarlove/K8s-project/assets/41736107/440e4623-003b-45c6-8f1e-43de0d7b6da8)


###### Why did I use Fargate over HPA ###### 

Using AWS Fargate over Horizontal Pod Autoscaling (HPA) is advantageous. 
Here are several reasons why I chose AWS Fargate instead of HPA:-

1. Serverless Computing: AWS Fargate is a service that lets you run containers without managing the servers they run on. It takes care of handling the infrastructure and scaling, so you can focus on deploying and running your containers. You don't need to manually adjust how much computing power is allocated to your containers, which is something you'd normally have to do with HPA.

2. Cost Efficiency: Fargate charges you based on how much CPU and memory your containers use. It automatically adjusts resources to match your workload, which helps you save money by not paying for more resources than you actually need.

3. Simplicity: Using Fargate makes managing Kubernetes easier because you don't have to set up or maintain EC2 instances (virtual servers). AWS handles all the infrastructure work like scaling and fixing problems behind the scenes.

4. Automatic Scaling: Fargate can increase or decrease the number of containers running your tasks based on how much CPU and memory they're using. You can set rules so it adjusts automatically as demand changes. This is similar to HPA, but it works at the level of individual tasks (containers) rather than entire groups of containers (pods in Kubernetes).

5. Isolation and Security: Each task you run with Fargate operates in its own separate environment. This setup provides better security and isolation compared to traditional Kubernetes pods, which share resources on the same server nodes.


##############################################################################################################################################################################
