# How to use LiteLLM to easily migrate to Amazon Bedrock
Some organisations have already built applications that work with OpenAI compatible API and would like to switch to Amazon Bedrock -- this implementation shows how you can do that without changing app code using LiteLLM.

## Architecture
The diagram below depicts the solution architecture. [LiteLLM](https://www.litellm.ai/) is used as a proxy to translate the API call originating from the app in OpenAI format to Bedrock format.

![architecture](docs/bedrock-litellm.drawio.png)

LiteLLM is deployed on Amazon EKS. If the app is hosted on the same cluster, it can access LiteLLM internally through Kubernetes `Service` of `type` `ClusterIP`. If the app is hosted outside the cluster, LiteLLM has to be exposed via a load balancer -- refer to [Exposing applications](https://www.eksworkshop.com/docs/fundamentals/exposing/) section of Amazon EKS workshop for guidance. This implementation assumes the app is hosted on the same cluster.

While LiteLLM is only used as a proxy in this implementation, it has several other features e.g. retry/fallback logic across multiple deployments, track spend & set budgets per project, etc.

## Implementation

**NOTE:** The implementation has been tested on Cloud9/Amazon Linux. Make sure to disable AWS managed temporary credentials and attach an IAM role with sufficient permissions.

### Prerequisites
- A domain that can be used for hosting Open WebUI, a web frontend that allows users to interact with LLMs; it will be used to test LiteLLM setup.
- A digital certificate in AWS Certificate Manager (ACM) for enabling TLS on Open WebUI

### Configure environment variables 
1. Configure environment variables; replace `<hostname>` with the hostname for OpenUI, and replace `<cert-arn>` with the ARN of the ACM certificate for Open WebUI

```sh
export HOSTNAME="<hostname>"
export CERTIFICATE_ARN="<cert-arn>"

echo "export HOSTNAME=${HOSTNAME}" | tee -a ~/.bash_profile
echo "export CERTIFICATE_ARN=${CERTIFICATE_ARN}" | tee -a ~/.bash_profile
```

```sh
export TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
export AWS_REGION=`curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region`
export CLUSTER_NAME="litellm-demo"

echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export CLUSTER_NAME=${CLUSTER_NAME}" | tee -a ~/.bash_profile
```
### Install Kubernetes tools

2. Install eksctl:
```sh
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```

3. Install kubectl:
```sh
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
```

4. Install yq:
```sh
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```

5. Install Helm:
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

6. Install envsubst:
```sh
curl -L https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-`uname -s`-`uname -m` -o envsubst
chmod +x envsubst
sudo mv envsubst /usr/local/bin
```

### Create an EKS cluster
7. Create cluster:
```sh
envsubst < bedrock-litellm/eksctl/cluster-config.yaml | eksctl create cluster -f -
```

### Install LiteLLM

8. Clone LiteLLM
```sh
git clone https://github.com/BerriAI/litellm.git
```

9. Create an IAM OIDC provider for the cluster to be able to use [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) (required for granting IAM permissions to LiteLLM to be able to invoke Bedrock models):
```sh
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve
```
10. Create IRSA dependencies for LiteLLM

First, create IAM policy:
```sh
sed -i "s/AWS_REGION/$AWS_REGION/g" bedrock-litellm/iam/litellm-bedrock-policy.json
export LITELLM_BEDROCK_IAM_POLICY_ARN=$(aws iam create-policy \
   --policy-name litellm-bedrock-policy \
   --policy-document file://bedrock-litellm/iam/litellm-bedrock-policy.json \
   --output text \
   --query "Policy.Arn")
echo "export LITELLM_BEDROCK_IAM_POLICY_ARN=${LITELLM_BEDROCK_IAM_POLICY_ARN}" | tee -a ~/.bash_profile
```

Then, create IRSA setup:
```sh
eksctl create iamserviceaccount \
    --name litellm-sa \
    --cluster $CLUSTER_NAME \
    --role-name AmazonEKS_LiteLLM_Role \
    --attach-policy-arn $LITELLM_BEDROCK_IAM_POLICY_ARN \
    --approve
```
11. Install LiteLLM

**NOTE:** LiteLLM helm chart is currently BETA, hence K8s manifests were used for installation. The snippet below will be changed once the helm chart is GAed.

```sh
yq -i '.spec.template.spec.serviceAccount= "litellm-sa"' litellm/deploy/kubernetes/kub.yaml
yq -i 'del(.spec.template.spec.containers[0].env[] | select(.name == "DATABASE_URL") )' litellm/deploy/kubernetes/kub.yaml
yq -i '.spec.type= "ClusterIP"' litellm/deploy/kubernetes/service.yaml

kubectl create configmap litellm-config --from-file=bedrock-litellm/litellm/proxy_config.yaml
kubectl apply -f litellm/deploy/kubernetes/kub.yaml
kubectl apply -f litellm/deploy/kubernetes/service.yaml
```

12. Verify LiteLLM

**NOTES:** Before you proceed with the steps below, make sure of the following:
- Acess to Bedrock models is enabled by following the steps in [this doc page](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html#model-access-add).
- LiteLLM pods are up and running

```sh
kubectl run curl --image=curlimages/curl --rm -it -- /bin/sh
curl --location 'http://litellm-service.default.svc.cluster.local:4000/chat/completions' \
    --header 'Content-Type: application/json' \
    --header 'Authorization: Bearer sk-1234' \
    --data '{
    "model": "bedrock-llama3-8b-instruct-v1",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ]
}'
```
### Use Open WebUI to test the setup

[Open WebUI]() is a web frontend that allows users to interact with LLMs. It supports locally running LLMs using Ollama, and OpenAI-compatible remote endpoints. In this implementation, we are configuring a remote endpoint that points to LiteLLM to show how LiteLLM allows for accessing Bedrock through an OpenAI-compatible interface. 

13. Install EBS CSI driver (EBS volumes will be used to store Open WebUI state):

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

14. Install AWS Load Balancer Controller (AWS LBC) to expose Open WebUI:

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

15. Install Open WebUI:
```sh
helm repo add open-webui https://helm.openwebui.com/
helm repo update

helm upgrade \
    --install open-webui open-webui/open-webui \
    --namespace open-webui \
    --create-namespace \
    -f bedrock-litellm/helm/open-webui-private-values.yaml
```

The first user signing up will get admin access. So, initially, Open WebUI will be only accessible from within the cluster to securely create the first/admin user. Subsequent sign ups will be in pending state till they are approved by the admin user.

16. Use `kubectl port-forward` to allow access to Open WebUI from the machine used for installation:
```sh
kubectl port-forward service/open-webui -n open-webui  8080:80
```
If you are using Cloud9, you can access Open WebUI by clicking "Preview" (top bar), then "Preview Running Application".

17. Sign-up (remember, first signed up user get admin access), then go to User icon at top right, settings, admin settings, connections, then edit OpenAI API to be as follows:

```sh
http://litellm-service.default.svc.cluster.local:4000/v1
```

Click on "Verify connection" button to make sure connectivity is in-place, then save. You should be able to see three of the Bedrock models available in Open WebUI as depicted in the screenshot below:

![architecture](docs/open-webui.png)

Now, we have the admin user created, we can make Open WebUI accessible publicly.

18. Update Open WebUI helm release to include `Ingress` object for exposing it:
```sh
envsubst < bedrock-litellm/helm/open-webui-public-values.yaml | helm upgrade \
    open-webui open-webui/open-webui \
    --namespace open-webui \
    -f -
```

19. Extract Open WebUI URL:
```sh
kubectl -n open-webui get ingress open-webui  -o jsonpath='{.status.loadBalancer.ingress[*].hostname}'
```

20. Add a CNAME record for `<hostname>` (check prerequisities section) that points to the ALB host name, then access Open WebUI using `<hostname>`.

**NOTE:** ELB needs a minutes or so to complete the target registration; if the URL above did not work for you, wait for a few seconds for the registration to get completed.

Edit `litellm/proxy_config.yaml`, update the IAM policy `litellm-bedrock-policy.json`, and enable access through the Bedrock console to add more Bedrock models on LiteLLM.

## Code Changes for OpenAI to Amazon Bedrock Migration

With LiteLLM successfully deployed onto Amazon EKS and proxying requests to Amazon Bedrock, you can choose to migrate from OpenAI with minimal code changes.

Requests from your applications that do not originate from Open WebUI can be modified by updating your OpenAI base endpoint to point to your ALB DNS name. This is similar to the change we made in step 17, updating an OpenAI endpoint to point to your LiteLLM service, this time to the ALB host name, or your CNAME record for <litellm-hostname> (check prerequisities section) that points to the ALB host name.

1. Update your application's OpenAI API endpoint to point to your <litellm-hostname>.

```python
import openai

openai.api_base = {"your-litellm-hostname"}
openai.api_key = {"your-open-ai-api-key"}

# Your existing OpenAI code remains unchanged
response = openai.Completion.create(
model="text-davinci-003",
prompt="Translate the following English text to French: 'Hello, how are you?'"
)
```

2. Test and validate that your existing code and application work as expected, calling foundation models hosted on Amazon Bedrock via LiteLLM hosted on Amazon EKS. Best practices and considerations:

    1. Gradually migrate: Start by routing a small percentage of traffic through the LiteLLM proxy and gradually increase as you gain confidence.
    2. Monitor performance: Use Amazon CloudWatch to monitor the performance and AWS Cost Explorer to monitor the costs of your AWS usage, including Amazon Bedrock.
    3. Security: Ensure least privilege AWS Identity and Access Management (AWS IAM) roles and security groups are in place for your EKS cluster and Amazon Bedrock access.
    4. Scalability: Configure auto-scaling for your EKS nodes to handle varying loads.

## Clean-up
1. Uninstall Open WebUI:
```sh
helm uninstall open-webui --namespace open-webui
```

2. Uninstall LiteLLM
```sh
kubectl delete -f litellm/deploy/kubernetes/service.yaml
kubectl delete -f litellm/deploy/kubernetes/kub.yaml
kubectl delete configmap litellm-config
eksctl delete iamserviceaccount \
    --name litellm-sa \
    --cluster $CLUSTER_NAME
aws iam delete-policy --policy-arn $LITELLM_BEDROCK_IAM_POLICY_ARN
```

3. Uninstall AWS LBC
```sh
helm uninstall aws-load-balancer-controller --namespace kube-system
eksctl delete iamserviceaccount \
    --name aws-load-balancer-controller \
    --namespace=kube-system \
    --cluster $CLUSTER_NAME
aws iam delete-policy --policy-arn $AWS_LBC_IAM_POLICY_ARN
```

4. Uninstall EBS driver
```sh
helm uninstall aws-ebs-csi-driver \
    --namespace kube-system
eksctl delete iamserviceaccount \
    --name ebs-csi-controller-sa \
    --cluster $CLUSTER_NAME
```

5. Delete cluster
```sh
eksctl delete cluster --name $CLUSTER_NAME
```

6. Delete the CNAME DNS record for Open WebUI, and delete the ACM certiciate

## Security

See CONTRIBUTING for more information.

## License
This library is licensed under the MIT-0 License. See the LICENSE file.