# Routing

LiteLLM's routing feature allows for dynamic control over how requests are directed to LLMs deployed across multiple backends. The routing configuration is critical for optimizing load distribution, cost management, fallback strategies, and latency.

💡 Note: To apply and test different routing configurations (such as load balancing, fallback, rate limit-aware routing, or latency-based routing), follow these steps:

1. Apply kubeconfig to update the configuration. Use the corresponding config file from the liteLLMConfig folder for each routing scenario.
2. Patch the LiteLLM deployment to ensure the new configuration is loaded.
3. Retrieve the Application Load Balancer (ALB) URL and test the behavior using the appropriate API calls.

Key routing functionalities include:

 **Load Balancing**:
LiteLLM distributes requests across multiple model instances using various strategies such as round-robin(default), least busy etc.

Example - Round-robin Load Balancing:

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

**Fallbacks**:
In case of a failure from a primary model, LiteLLM automatically redirects the request to a fallback model. This ensures uninterrupted service even if a model or provider is down.

Example

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

For testing Make sure when you pass additional field called **mock_testing_fallbacks** as shown below

```bash
curl --location "http://$ALB_DNS_NAME/chat/completions" \
    --header 'Content-Type: application/json' \
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

**Rate Limit-Aware Routing**

LiteLLM can dynamically reroute requests if a model has exceeded its rate limit (requests per minute or tokens per minute). This prevents service disruption when models reach their capacity limits.

Example

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

**Latency Based Routing**

LiteLLM can prioritize routing based on model response times (latency). It Picks the deployment with the lowest response time by caching, and updating the response times for deployments based on when a request was sent and received from a deployment.

Example

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

For more details on routing, please refer to the [liteLLM](https://docs.litellm.ai/docs/routing) docs.