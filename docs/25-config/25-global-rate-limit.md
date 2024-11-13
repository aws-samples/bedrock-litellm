# Global rate limiting

LiteLLM allows you to apply requests per minute (RPM) and tokens per minute (TPM) limits globally across all users, teams, and models through the LiteLLM configuration file. These global limits ensure that traffic is controlled across all requests, regardless of individual user or team limits.

!!! note annotate "Note"
    You do not need to configure a database URL or use the LLM master key to apply this configuration, making it simpler for deployments where per-user tracking is not required.


## Sample configuration

```yaml
model_list:
  - model_name: claude-3
    litellm_params:
      model: bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0
      aws_region_name: $AWS_REGION
      rpm: 2 
      tpm: 200

router_settings:
  enable_pre_call_checks: true # 1. Enable pre-call checks
```

## Steps to configure

1. Use the configuration file `$BEDROCK_LITELLM_DIR/litellm/config/proxy_config_global_rate_limit.yaml`
1. To apply the new configuration, follow the steps outlined in [Apply configuration changes](./40-apply-config-changes.md).

## Steps to test
1. To test the global rate limit, make three or more API requests within one minute. After the second request, you should start receiving an error indicating that the limit has been reached. The limit will reset after one minute.
  ```bash
  curl --location "http://${LITELLM_HOSTNAME}/chat/completions" \
    --header 'Content-Type: application/json' \
    --data '{
      "model": "claude-3",
      "messages": [
        {
          "role": "user",
          "content": "what is amazon S3"
        }
      ]
    }'
  ```
  If the rate limit is exceeded, you should receive an error response similar to the one below:
  ```json
  {
    "error":
      {
        "message":"No deployments available for selected model, Try again in 60 seconds. Passed model=bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0. Try again in 60 seconds.",
        "type":"None",
        "param":"None",
        "code":"429"
      }
  }
  ```
