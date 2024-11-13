# Routing

LiteLLM's routing feature allows for dynamic control over how requests are directed to LLMs deployed across multiple backends. The routing configuration is critical for optimizing load distribution, cost management, fallback strategies, and latency.

## Steps to configure

To apply different routing configurations (such as load balancing, fallback, rate limit-aware routing, or latency-based routing), follow these steps:

1. Use one of the configuration files at `$BEDROCK_LITELLM_DIR/litellm/config/route/`
1. Follow the steps outlined in [Apply configuration changes](./40-apply-config-changes.md).

## Steps to test
1. Test by making a call to the chat API:
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

## Sample configurations

The sample configurations below demonstrate LiteLLM key routing functionalities.

### Load balancing
LiteLLM distributes requests across multiple model instances using various strategies such as round-robin(default), least busy etc.

```yaml
model_list:
  - model_name: claude-3
    litellm_params:
      model: bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0
      aws_region_name: $AWS_REGION
  
  - model_name: claude-3
    litellm_params:
      model: bedrock/anthropic.claude-3-haiku-20240307-v1:0
      aws_region_name: $AWS_REGION
```

### Fallbacks
In case of a failure from a primary model, LiteLLM automatically redirects the request to a fallback model. This ensures uninterrupted service even if a model or provider is down.

```yaml
model_list:
  - model_name: claude-3-sonnet
    litellm_params:
      model: bedrock/invalid
      aws_region_name: $AWS_REGION
  
  - model_name: claude-3-haiku
    litellm_params:
      model: bedrock/anthropic.claude-3-haiku-20240307-v1:0
      aws_region_name: $AWS_REGION

router_settings:
    enable_pre_call_checks: true # 1. Enable pre-call checks

litellm_settings:
  num_retries: 3 # retry call 3 times on each model_name (e.g. zephyr-beta)
  request_timeout: 10 # raise Timeout error if call takes longer than 10s. Sets litellm.request_timeout 
  fallbacks: [{"claude-3-sonnet": ["claude-3-haiku"]}] 
  allowed_fails: 3 # cooldown model if it fails > 1 call in a minute. 
  cooldown_time: 30 # how long to cooldown model if fails/min > allowed_fails
```

For testing, make sure when you pass additional field called `mock_testing_fallbacks` as shown below:
```bash
curl --location "http://${LITELLM_HOSTNAME}/chat/completions" \
    --header 'Content-Type: application/json' \
    --header 'Authorization: Bearer <your key>' \
    --data '{
    "model": "claude-3-sonnet",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ], "mock_testing_fallbacks": true
}'
```

### Rate limit-aware routing
LiteLLM can dynamically reroute requests if a model has exceeded its rate limit (requests per minute or tokens per minute). This prevents service disruption when models reach their capacity limits.

```yaml
model_list:
  - model_name: claude-3-sonnet
    litellm_params: 
      model: bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0
      aws_region_name: $AWS_REGION
    tpm: 2000
    rpm: 10
  - model_name: claude-3-haiku
    litellm_params: 
      model: bedrock/anthropic.claude-3-haiku-20240307-v1:0
      aws_region_name: $AWS_REGION
    tpm: 10000
    rpm: 1

router_settings:
  routing_strategy: usage-based-routing-v2
  enable_pre_call_check: true
```

In this configuration, Claude Sonnet is limited to 10 requests per minute and 2000 tokens per minute. When exceeding these limits, LiteLLM filters out the deployment, and routes to the deployment with the lowest TPM usage for that minute.

### Latency-based routing

LiteLLM can prioritize routing based on model response times (latency). It Picks the deployment with the lowest response time by caching, and updating the response times for deployments based on when a request was sent and received from a deployment.

```yaml
model_list:
  - model_name: claude-3
    litellm_params: 
      model: bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0
      aws_region_name: $AWS_REGION
  
  - model_name: claude-3
    litellm_params: 
      model: bedrock/anthropic.claude-3-haiku-20240307-v1:0
      aws_region_name: $AWS_REGION

router_settings:
  routing_strategy: latency-based-routing"
  enable_pre_call_check: true
```

For more details on routing, please refer to the [LiteLLM](https://docs.litellm.ai/docs/routing) docs.