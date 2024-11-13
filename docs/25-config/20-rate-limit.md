LiteLLM provides rate-limiting capabilities to control the number of requests or tokens that can be processed by a model over a specific time. This feature allows you to manage traffic, control costs, and ensure fairness by restricting access based on requests per minute (RPM) or tokens per minute (TPM).

Rate limits can be applied:

1. Per API key
2. Per user
3. Per team
4. Per specific models

You can specify both rpm_limit (requests per minute) and tpm_limit (tokens per minute) for models, users, or teams in your configuration.

Here is an example of restricting RPM and TPM for a certain user

1. Create user with RPM and TPM values.

You can create a user and define the limits for RPM and TPM by sending a request to the user creation API. This will ensure that the user is rate-limited accordingly.

```bash
curl --location 'http://$ALB_DNS_NAME/user/new' \
--header 'Authorization: Bearer <your key>' \
--header 'Content-Type: application/json' \
--data '{
    "user_id": "test_user_1", 
    "max_parallel_requests": 10, 
    "tpm_limit": 20, 
    "rpm_limit": 2
}'

```

   2. You should get a '**key**'in the response header. This will server as master key while making chat request. for example lets say value returned is 'sk-1234567'

   3. Using the master key obtained in step 2, you can make a request to the chat API. Ensure the correct model and message are passed, and the authorization header includes the master key:

```bash
curl --location "http://$ALB_DNS_NAME/chat/completions" \
    --header 'Content-Type: application/json' \
    --header 'Authorization: Bearer sk-1234567' \
    --data '{
    "model": "claude-3",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ]
}'

```

4. After making more than 2 requests, you should start receiving an error response indicating that the RPM limit has been reached. Here is an example of the error response you might see:

```json
{
    "error": {
        "message": "Max parallel request limit reached. Hit limit for api_key: xxx. tpm_limit: 20, current_tpm 46, rpm_limit: 2, current rpm 1",
        "type": "None",
        "param": "None",
        "code": "429"
    }
}
```

Similarly you can perform other tests on TPM as well.

Please note above test is for applying rate limit on user level. If you want to test different configurations on teams, organization etc, you can follow liteLLM documentation [here](https://docs.litellm.ai/docs/proxy/users)


**Steps to configure**

1. This file is present in $BEDROCK_LITELLM_DIR/litellm/config/litellm-config-rateLimitAwareRouting
2. Follow the steps outlined in [Apply configuration changes](./40-apply-config-changes.md).

**Global Rate Limit**

LiteLLM allows you to apply requests per minute (RPM) and tokens per minute (TPM) limits globally across all users, teams, and models through the LiteLLM configuration file. These global limits ensure that traffic is controlled across all requests, regardless of individual user or team limits.

Note: You do not need to configure a database URL or use the LLM master key to apply this configuration, making it simpler for deployments where per-user tracking is not required.

**Example Config**

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

**Steps to configure**

1. This file is present in $BEDROCK_LITELLM_DIR/litellm/config/litellm-config-globalRateLimit.yaml
2. Follow the steps outlined in [Apply configuration changes](./40-apply-config-changes.md).
3. To test the global rate limit, make three or more API requests within one minute. After the second request, you should start receiving an error indicating that the limit has been reached. The limit will reset after one minute.

```bash
curl --location "http://$ALB_DNS_NAME/chat/completions" \
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
{"error":{"message":"No deployments available for selected model, Try again in 60 seconds. Passed model=bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0. Try again in 60 seconds.","type":"None","param":"None","code":"429"}}%     
```
