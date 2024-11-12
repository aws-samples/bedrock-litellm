# Apply configuration changes

In order to apply LiteLLM configuration changes in ConfigMap, follow the steps below to ensure the deployment is patched, thereby forcing a pod restart to reflect latest changes:

!!! note annotate "Note"
    If  the configuration file contains environment variables (e.g., `$AWS_REGION`), use the `envsubst` command to replace the variables with their actual values, before updating the ConfigMap. See an example below.

!!! tip annotate "Note"
    Replace the filename below to reflect your file with configuration update i.e. `$BEDROCK_LITELLM_DIR/litellm/config/proxy_config_model_alias.yaml` to be replaced with your relevant file that contains the updated configuration.
    
1. Run the following command to replace the environment variables with their values.
```sh
envsubst < $BEDROCK_LITELLM_DIR/litellm/config/proxy_config_model_alias.yaml > /tmp/proxy_config_model_alias.yaml
```
1. Run the following command to update the configMap with latest configuration from a file.
```sh
kubectl create configmap litellm-config --from-file=litellm-config.yaml=/tmp/proxy_config_model_alias.yaml --dry-run=client -o yaml | kubectl apply -f -
```
1. Patch the deployment to reflect the changes. After updating the ConfigMap, the pods won’t automatically reload the configuration. You need to patch the deployment to force a pod restart, ensuring that the new configuration is picked up.
```sh
kubectl patch deployment litellm-deployment -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"configmap-update-timestamp\":\"$(date +'%s')\"}}}}}"
```
1. Once the pods are restarted, verify the new configuration is applied by checking the logs.
