# Code Changes for OpenAI to Amazon Bedrock Migration

With LiteLLM successfully deployed onto Amazon EKS and proxying requests to Amazon Bedrock, you can choose to migrate from OpenAI with minimal code changes.

Requests from your applications that do not originate from Open WebUI can be modified by updating your OpenAI base endpoint to point to your ALB DNS name. This is similar to the change we made in step 17, updating an OpenAI endpoint to point to your LiteLLM service, this time to the ALB host name, or your CNAME record for <litellm-hostname> (check prerequisities section) that points to the ALB host name.

1. Update your application's OpenAI API endpoint to point to your <litellm-hostname>.

    ```python
    import openai

    openai.api_base = {"your-litellm-hostname"}
    openai.api_key = {"your-open-ai-api-key"}

    # Your existing OpenAI code remains unchanged
    response = openai.Completion.create(
    model="text-davinci-003",
    prompt="Translate the following English text to French: 'Hello, how are you?'"
    )
    ```

1. Test and validate that your existing code and application work as expected, calling foundation models hosted on Amazon Bedrock via LiteLLM hosted on Amazon EKS. Best practices and considerations:

    1. Gradually migrate: Start by routing a small percentage of traffic through the LiteLLM proxy and gradually increase as you gain confidence.
    2. Monitor performance: Use Amazon CloudWatch to monitor the performance and AWS Cost Explorer to monitor the costs of your AWS usage, including Amazon Bedrock.
    3. Security: Ensure least privilege AWS Identity and Access Management (AWS IAM) roles and security groups are in place for your EKS cluster and Amazon Bedrock access.
    4. Scalability: Configure auto-scaling for your EKS nodes to handle varying loads.
