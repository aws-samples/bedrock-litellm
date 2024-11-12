# (Optional) Connect Open WebUI to LiteLLM

[Open WebUI]() is a web frontend that allows users to interact with LLMs. It supports locally running LLMs using Ollama, and OpenAI-compatible remote endpoints. In this implementation, we are configuring a remote endpoint that points to LiteLLM to show how LiteLLM allows for accessing Bedrock through an OpenAI-compatible interface. 

## Pre-requisites
- A domain that can be used for hosting Open WebUI, a web frontend that allows users to interact with LLMs; it will be used to test LiteLLM setup.
- A digital certificate in AWS Certificate Manager (ACM) for enabling TLS on Open WebUI


## Open WebUI deployment
1. Configure environment variables; replace `<open-webui-hostname>`, `<open-webui-cert-arn>` with the corresponding hostnames and ACM certificates ARN.

    ```sh
    export OPEN_WEBUI_HOSTNAME="<open-webui-hostname>"
    export OPEN_WEBUI_CERTIFICATE_ARN="<open-webui-cert-arn>"

    echo "export OPEN_WEBUI_HOSTNAME=${OPEN_WEBUI_HOSTNAME}" | tee -a ~/.bash_profile
    echo "export OPEN_WEBUI_CERTIFICATE_ARN=${OPEN_WEBUI_CERTIFICATE_ARN}" | tee -a ~/.bash_profile
    ```

1. Install Open WebUI:
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

1. Use `kubectl port-forward` to allow access to Open WebUI from the machine used for installation:
    ```sh
    kubectl port-forward service/open-webui -n open-webui  8080:80
    ```
    
    If you are using Cloud9, you can access Open WebUI by clicking "Preview" (top bar), then "Preview Running Application".

1. Sign-up (remember, first signed up user get admin access), then go to User icon at top right, settings, admin settings, connections, then edit OpenAI API to be as follows:

    ```sh
    http://litellm-service.default.svc.cluster.local:4000/v1
    ```

    Click on "Verify connection" button to make sure connectivity is in-place, then save. You should be able to see three of the Bedrock models available in Open WebUI as depicted in the screenshot below:

    ![architecture](../open-webui.png)

    Now, we have the admin user created, we can make Open WebUI accessible publicly.

1. Update Open WebUI helm release to include `Ingress` object for exposing it:
    ```sh
    envsubst < $BEDROCK_LITELLM_DIR/helm/open-webui-public-values.yaml | helm upgrade \
        open-webui open-webui/open-webui \
        --namespace open-webui \
        -f -
    ```
    
    !!! note annotate "Note"
        ELB needs a minute or so to complete the target registration; if the URL above did not work for you, wait for a few seconds for the registration to get completed.


1. Extract Open WebUI URL:
    ```sh
    kubectl -n open-webui get ingress open-webui  -o jsonpath='{.status.loadBalancer.ingress[*].hostname}'
    ```

1. Add a CNAME record for `<open-webui-hostname>` (check prerequisities section) that points to the ALB host name, then access Open WebUI using `<open-webui-hostname>`.


1. Edit `litellm/proxy_config.yaml`, update the IAM policy `litellm-bedrock-policy.json`, and enable access through the Bedrock console to add more Bedrock models on LiteLLM.