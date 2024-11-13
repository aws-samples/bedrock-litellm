# Model aliases

Model aliases help you define a user-friendly name for a model or group of models, abstracting away the actual model name used by the model provider. LiteLLM provides the ability to use model aliases, which allow you to present a simplified or user-friendly model name to your end-users while invoking a different, more specific model name on the backend. This is useful when you want to abstract Bedrock model ids from clients or when dealing with multiple versions of models.

For example, you might display the model name `claude-3` to the end-user while internally calling `bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0` on the backend.

In the configuration file `proxy_config_model_alias.yaml`, the `model_name` parameter (e.g., `claude-3`) is the user-facing name, while the `litellm_params.model` parameter contains the actual backend model id (e.g., `bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0`) and any additional parameters needed for model configuration.

## Sample configuration

```yaml
model_list:
  - model_name: claude-3
    litellm_params:
      model: bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0
      aws_region_name: $AWS_REGION
```

## Steps to configure

1. Use the configuration file `$BEDROCK_LITELLM_DIR/litellm/config/proxy_config_model_alias.yaml`
1. To apply the new configuration, follow the steps outlined in [Apply configuration changes](./40-apply-config-changes.md).

## Steps to test
1. Test the model alias by invoking the user-facing model name as shown
```sh
curl --location "https://${LITELLM_HOSTNAME}/chat/completions" \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer <your key>' \
  --data '{
    "model": "claude-3",
    "messages": [
      {
        "role": "user",
        "content": "what is Amazon S3"
      }
    ]
  }'
```
