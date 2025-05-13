# Install LiteLLM
1. Clone LiteLLM repo
```sh
git clone https://github.com/BerriAI/litellm.git
```

1. Save LiteLLM directory in an environment variable
```sh
export LITELLM_DIR=$PWD/litellm
echo "export LITELLM_DIR=${LITELLM_DIR}" | tee -a ~/.bash_profile
```

1. (Optional) If you plan to use LiteLLM to connect to SageMaker, retrieve the endpoint name.

    First, find the endpoint that you wish to use. If you wish to deploy a new foundation model endpoint use SageMaker JumpStart, please refer to [JumpStart foundation model usage](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-use.html). To list the available endpoints:

    ```sh
    aws sagemaker list-endpoints
    ```

    Then, save the endpoint name as an environment variable:
    ```sh
    export ENDPOINT_NAME="your-endpoint-name"
    echo "export ENDPOINT_NAME=${ENDPOINT_NAME}" | tee -a ~/.bash_profile
    ```

1. Create IRSA dependencies for LiteLLM

    First, create the IAM policy.

    !!! note annotate "Note"
        If you are also connecting to an endpoint running on Amazon SageMaker, replace the next command below with:
            ```sh
            envsubst < $BEDROCK_LITELLM_DIR/iam/litellm-bedrock-and-sagemaker-policy.json > /tmp/litellm-bedrock-policy.json
            ```

    Running the following cell only if you are not connection to a SageMaker Endpoint, otherwise run the above cell:
    ```sh
    envsubst < $BEDROCK_LITELLM_DIR/iam/litellm-bedrock-policy.json > /tmp/litellm-bedrock-policy.json
    ```

    Create the policy:
    ```sh
    export LITELLM_BEDROCK_IAM_POLICY_ARN=$(aws iam create-policy \
    --policy-name litellm-bedrock-policy \
    --policy-document file:///tmp/litellm-bedrock-policy.json \
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

1. Install LiteLLM

    !!! warning annotate "Note"
        LiteLLM helm chart is currently BETA, hence K8s manifests were used for installation. The snippet below will be changed once the helm chart is GAed.

    !!! warning annotate "Note"
        Make sure to change `LITELLM_MASTER_KEY` in `$LITELLM_DIR/deploy/kubernetes/kub.yaml` to a random string rather than using the default API key, especially if you will expose LiteLLM endpoint externally.

    ```sh
    yq -i '.spec.template.spec.serviceAccount= "litellm-sa"' litellm/deploy/kubernetes/kub.yaml
    yq -i 'del(.spec.template.spec.containers[0].env[] | select(.name == "DATABASE_URL") )' litellm/deploy/kubernetes/kub.yaml
    yq -i '.spec.type= "ClusterIP"' litellm/deploy/kubernetes/service.yaml
    ```
    !!! note annotate "Note"
        If you are also connect to an endpoint running on Amazon SageMaker, run next the following cell before creating the configmap:
            ```sh
            envsubst < $BEDROCK_LITELLM_DIR/litellm/config/proxy_config_with_sagemaker_models.yaml > $BEDROCK_LITELLM_DIR/litellm/config/proxy_config.yaml
            ```

    Create the configmap:
    ```sh
    kubectl create configmap litellm-config --from-file=$BEDROCK_LITELLM_DIR/litellm/config/proxy_config.yaml
    ```

    Apply the changes:
    ```sh
    kubectl apply -f $LITELLM_DIR/deploy/kubernetes/kub.yaml
    kubectl apply -f $LITELLM_DIR/deploy/kubernetes/service.yaml
    ```

1. Allow acess to Bedrock models by following the steps in [this doc page](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html#model-access-add).

1. Ensure that LiteLLM pods are up and running

1. Verify LiteLLM

    ```sh
    kubectl run curl --image=curlimages/curl --rm -it -- /bin/sh
    curl --location "http://litellm-service.default.svc.cluster.local:4000/chat/completions" \
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

    (Optional) If you configured a SageMaker Endpoint, you can also query this, for example:

    ```sh
    curl --location "http://litellm-service.default.svc.cluster.local:4000/chat/completions" \
        --header 'Content-Type: application/json' \
        --header 'Authorization: Bearer sk-1234' \
        --data '{
        "model": "sagemaker-model",
        "messages": [
            {
            "role": "user",
            "content": "Write me a haiku about Mount Fuji"
            }
        ]
    }'
    ```