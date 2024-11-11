# Prerequisites
If you will use Open WebUI to test LiteLLM configurations, the following prerequisites apply:

- A domain that can be used for hosting Open WebUI, a web frontend that allows users to interact with LLMs; it will be used to test LiteLLM setup.
- A digital certificate in AWS Certificate Manager (ACM) for enabling TLS on Open WebUI

If you will expose LiteLLM outside the EKS cluster, the following prerequisites apply:

- A domain that can be used for hosting LiteLLM and exposing it externally through public endpoint.
- A digital certificate in AWS Certificate Manager (ACM) for enabling TLS on LiteLLM

