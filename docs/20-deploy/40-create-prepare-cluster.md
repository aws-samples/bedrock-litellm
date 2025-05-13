# Create and prepare an EKS cluster

1. Create cluster:
```sh
envsubst < $BEDROCK_LITELLM_DIR/eksctl/cluster-config.yaml | eksctl create cluster -f -
```

1. Create an IAM OIDC provider for the cluster to be able to use [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) (required for granting IAM permissions to LiteLLM to be able to invoke Bedrock models):
```sh
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve
```

1. (Optional) Install AWS Load Balancer Controller (AWS LBC):

    !!! note annotate "Note"
        Install AWS Load Balancer Controller if you are planning to expose LiteLLM or one of the clients on ELB.

    First, create the IAM policy:

    ```sh
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

    export AWS_LBC_IAM_POLICY_ARN=$(aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json \
    --output text \
    --query "Policy.Arn")
    echo "export AWS_LBC_IAM_POLICY_ARN=${AWS_LBC_IAM_POLICY_ARN}" | tee -a ~/.bash_profile
    ```

    Then, create IRSA setup:
    ```sh
    eksctl create iamserviceaccount \
        --cluster $CLUSTER_NAME \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKS_LoadBalancerController_Role \
        --attach-policy-arn $AWS_LBC_IAM_POLICY_ARN \
        --approve
    ```
    Then, install AWS LBC helm chart:
    ```sh
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace kube-system \
    --set clusterName=$CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller 
    ```


1. (Optional) Install EBS CSI driver (EBS volumes will be used to store Open WebUI state):

    !!! note annotate "Note"
        Install EBS CSI driver if you are planning to use Open WebUI as it depends on EBS volumes for storing its state.

    First, create IRSA dependencies:
    ```sh
    eksctl create iamserviceaccount \
        --name ebs-csi-controller-sa \
        --namespace kube-system \
        --cluster $CLUSTER_NAME \
        --role-name AmazonEKS_EBS_CSI_DriverRole \
        --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
        --approve
    ```
    Then, install EBS CSI driver helm chart:
    ```sh
    helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
    helm repo update
    helm upgrade --install aws-ebs-csi-driver \
        --namespace kube-system \
        --set controller.serviceAccount.create=false \
        aws-ebs-csi-driver/aws-ebs-csi-driver
    ```
