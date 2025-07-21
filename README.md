# Deploy this funny Game app with AWS EKS.

1. prerequisites:

      i) You Should have kubectl, eksctl, AWS CLI

      ii) you can install this from official documentation and need to configure your aws.

2. Install EKS:

      i) Install using Fargate:
   
       eksctl create cluster --name demo-cluster --region us-east-1 --fargate

      ii) At the end You can delete your cluster uaing:

       eksctl delete cluster --name demo-cluster --region ap-south-1
   
4. Now you need to download kube-configfile for configure your local kubectl tool to connect to your EKS cluster

       aws eks update-kubeconfig --name demo-cluster --region ap-south-1

5. Now we deploy our application pod using the deployment

       eksctl create fargateprofile \
        --cluster demo-cluster \
        --region ap-south-1 \
        --name alb-sample-app \
        --namespace game-2048

6. Deploy the deployment, service and Ingress

        kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

7. Now we need to enable IAM OIDC connecter beacuse EKS clusters can assign AWS IAM permissions to Kubernetes pods — but only if OIDC is enabled.

   7.1) OIDC (OpenID Connect) is a standard identity provider that allows EKS to:

      i) Connect IAM roles to Kubernetes service accounts

      ii) So that individual pods can access AWS services securely using their own IAM role

       eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

8. Now You need to Download IAM Policy for AWS Load Balancer Controller

      i) When you deploy the AWS Load Balancer Controller in EKS, it needs permissions to manage AWS resources like:

      a) Creating ALBs

      b) Registering targets (pods)
 
      c) Managing security groups, listeners, etc.
   
   But Kubernetes doesn’t have IAM permissions by default. That’s where the IAM policy comes in.

       curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

     ii) Create IAM Policy

           aws iam create-policy \
            --policy-name AWSLoadBalancerControllerIAMPolicy \
            --policy-document file://iam_policy.json

     iii) Create IAM Role

           eksctl create iamserviceaccount \
               --cluster=<your-cluster-name> \
               --namespace=kube-system \
               --name=aws-load-balancer-controller \
               --role-name AmazonEKSLoadBalancerControllerRole \
               --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
               --approve

 9.  Deploy ALB controller:

        i) Add helm repo

            helm repo add eks https://aws.github.io/eks-charts

        ii) Update the repo

            helm repo update eks

        iii) Install:

             helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
                  --set clusterName=<your-cluster-name> \
                  --set serviceAccount.create=false \
                  --set serviceAccount.name=aws-load-balancer-controller \
                  --set region=<your-region> \
                  --set vpcId=<your-vpc-id>

        iv)  Verify that the deployments are running.

                kubectl get deployment -n kube-system aws-load-balancer-controller

 




















   
