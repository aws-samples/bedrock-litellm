# Rate limiting

LiteLLM provides rate-limiting capabilities to control the number of requests or tokens that can be processed by a model over a specific time. This feature allows you to manage traffic, control costs, and ensure fairness by restricting access based on requests per minute (RPM) or tokens per minute (TPM).

Rate limits can be applied:

1. Per API key
2. Per user
3. Per team
4. Per specific models

You can specify both rpm_limit (requests per minute) and tpm_limit (tokens per minute) for models, users, or teams in your configuration.

## Steps to configure

1. Create user with RPM and TPM values. You can create a user and define the limits for RPM and TPM by sending a request to the user creation API. This will ensure that the user is rate-limited accordingly.
  ```bash
  curl --location 'http://${LITELLM_HOSTNAME}/user/new' \
  --header 'Authorization: Bearer <your key>' \
  --header 'Content-Type: application/json' \
  --data '{
      "user_id": "test_user_1", 
      "max_parallel_requests": 10, 
      "tpm_limit": 20, 
      "rpm_limit": 2
  }'
  ```
  You should get a `key` in the response header. This will serve as a master key while making chat request. For example, lets say value returned is `sk-1234567`.

## Steps to test

1. Using the master key obtained in the previous step, you can make a request to the chat API. Ensure the correct model and message are passed, and the authorization header includes the master key:
  ```bash
  curl --location "http://${LITELLM_HOSTNAME}/chat/completions" \
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
1. After making more than 2 requests, you should start receiving an error response indicating that the RPM limit has been reached. Here is an example of the error response you might see:
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
  
  Please note the above test is for applying rate limit on user level. If you want to test different configurations on teams, organization etc, you can follow LiteLLM documentation [here](https://docs.litellm.ai/docs/proxy/users).
