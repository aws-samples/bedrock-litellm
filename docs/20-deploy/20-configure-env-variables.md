# Configure environment variables

!!! note annotate "Note"
    The steps in the following sections have been tested on Cloud9/Amazon Linux. Make sure to disable AWS managed temporary credentials and attach an IAM role with sufficient permissions.

1. Configure environment variables

```sh
export TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
export AWS_REGION=`curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region`
export ACCOUNT_ID=`curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/identity-credentials/ec2/info | jq -r '.AccountId'`
export CLUSTER_NAME="litellm-demo"

echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export CLUSTER_NAME=${CLUSTER_NAME}" | tee -a ~/.bash_profile
```
