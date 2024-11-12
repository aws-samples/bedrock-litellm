# (Optional) Expose LiteLLM

## Pre-requisites
- A domain that can be used for hosting LiteLLM and exposing it externally through public endpoint.
- A digital certificate in AWS Certificate Manager (ACM) for enabling TLS on LiteLLM

## Expose LiteLLM
1. Configure environment variables; replace `<litellm-hostname>`, `<litellm-cert-arn>` with the corresponding hostnames and ACM certificates ARN.
    ```sh
    export LITELLM_HOSTNAME="<litellm-hostname>"
    export LITELLM_CERTIFICATE_ARN="<litellm-cert-arn>"

    echo "export LITELLM_HOSTNAME=${LITELLM_HOSTNAME}" | tee -a ~/.bash_profile
    echo "export LITELLM_CERTIFICATE_ARN=${LITELLM_CERTIFICATE_ARN}" | tee -a ~/.bash_profile
    ```

1. Apply LiteLLM ingress
    ```sh
    envsubst < $BEDROCK_LITELLM_DIR/litellm/ingress.yaml | kubectl apply -f -
    ```

    !!! note annotate "Note"
        ELB needs a minute or so to complete the target registration; if the URL above did not work for you, wait for a few seconds for the registration to get completed.

1. Extract LiteLLM URL:
    ```sh
    kubectl get ingress litellm-ingress  -o jsonpath='{.status.loadBalancer.ingress[*].hostname}'
    ```

1. Add a CNAME record for `<litellm-hostname>` (check prerequisities section) that points to the ALB host name, then access LiteLLM using `<litellm-hostname>`.


1. Verify LiteLLM through external endpoint

    ```sh
    curl --location "https://${LITELLM_HOSTNAME}/chat/completions" \
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
