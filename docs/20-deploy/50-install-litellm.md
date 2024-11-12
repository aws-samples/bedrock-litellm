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

1. Create IRSA dependencies for LiteLLM

    First, create IAM policy:
    ```sh
    sed -i "s/AWS_REGION/$AWS_REGION/g" $BEDROCK_LITELLM_DIR/iam/litellm-bedrock-policy.json
    export LITELLM_BEDROCK_IAM_POLICY_ARN=$(aws iam create-policy \
    --policy-name litellm-bedrock-policy \
    --policy-document file://$BEDROCK_LITELLM_DIR/iam/litellm-bedrock-policy.json \
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
        Make sure to change `LITELLM_MASTER_KEY` in `$LITELLM_DIR/deploy/kubernetes/kub.yaml` to a random string rather than using the default API key, specially if you will expose LiteLLM endpoint externally.

    ```sh
    yq -i '.spec.template.spec.serviceAccount= "litellm-sa"' $LITELLM_DIR/deploy/kubernetes/kub.yaml
    yq -i 'del(.spec.template.spec.containers[0].env[] | select(.name == "DATABASE_URL") )' $LITELLM_DIR/deploy/kubernetes/kub.yaml
    yq -i '.spec.type= "ClusterIP"' $LITELLM_DIR/deploy/kubernetes/service.yaml

    kubectl create configmap litellm-config --from-file=$BEDROCK_LITELLM_DIR/litellm/proxy_config.yaml
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
