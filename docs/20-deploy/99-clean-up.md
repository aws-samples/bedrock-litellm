1. Uninstall Open WebUI:
```sh
helm uninstall open-webui --namespace open-webui
```

2. Uninstall LiteLLM
```sh
kubectl delete -f $BEDROCK_LITELLM_DIR/litellm/ingress.yaml
kubectl delete -f $LITELLM_DIR/deploy/kubernetes/service.yaml
kubectl delete -f $LITELLM_DIR/deploy/kubernetes/kub.yaml
kubectl delete configmap litellm-config
eksctl delete iamserviceaccount \
    --name litellm-sa \
    --cluster $CLUSTER_NAME
aws iam delete-policy --policy-arn $LITELLM_BEDROCK_IAM_POLICY_ARN
```

3. Uninstall AWS LBC
```sh
helm uninstall aws-load-balancer-controller --namespace kube-system
eksctl delete iamserviceaccount \
    --name aws-load-balancer-controller \
    --namespace=kube-system \
    --cluster $CLUSTER_NAME
aws iam delete-policy --policy-arn $AWS_LBC_IAM_POLICY_ARN
```

4. Uninstall EBS driver
```sh
helm uninstall aws-ebs-csi-driver \
    --namespace kube-system
eksctl delete iamserviceaccount \
    --name ebs-csi-controller-sa \
    --cluster $CLUSTER_NAME
```

5. Delete cluster
```sh
eksctl delete cluster --name $CLUSTER_NAME
```

6. Delete the CNAME DNS records and the ACM certiciates used for LiteLLM and Open WebUI
