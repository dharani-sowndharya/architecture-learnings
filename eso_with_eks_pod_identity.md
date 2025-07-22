# External Secrets Operator (ESO) with EKS Pod Identity

This guide demonstrates how to set up External Secrets Operator (ESO) with EKS Pod Identity to securely retrieve secrets from AWS Secrets Manager without using static credentials.

## Overview

EKS Pod Identity provides a secure way for Kubernetes pods to authenticate with AWS services by mapping Kubernetes ServiceAccounts to AWS IAM roles. This eliminates the need for static AWS credentials and provides automatic credential rotation.

## Architecture Flow

```mermaid
graph TD
    subgraph "EKS Cluster"
        subgraph "Kube-system Namespace"
            PIA[Pod Identity Agent<br/>DaemonSet]
        end
        
        subgraph "External-secrets Namespace"
            SA[ServiceAccount<br/>external-secrets]
            ESO[ESO Controller<br/>Pod]
            CSS[ClusterSecretStore<br/>aws-secrets-manager]
            ES[ExternalSecret<br/>app-secret]
            K8sSecret[Kubernetes Secret<br/>app-credentials]
        end
        
        subgraph "App Namespace"
            AppPod[Application Pod]
        end
    end
    
    subgraph "AWS"
        subgraph "EKS Service"
            PIAS[Pod Identity<br/>Association]
        end
        
        subgraph "IAM"
            IAMRole[IAM Role<br/>YourESRole]
            IAMPolicy[IAM Policy<br/>SecretsManager Access]
        end
        
        subgraph "AWS Secrets Manager"
            SecretASM[Secret<br/>your-secret-name]
        end
    end
    
    %% Configuration Relationships (dashed lines)
    PIAS -.->|configures| SA
    PIAS -.->|maps to| IAMRole
    IAMRole -.->|has policy| IAMPolicy
    CSS -.->|auth config| SA
    
    %% Numbered Flow Steps
    ESO -->|1| ES
    ES -->|2| CSS
    CSS -->|3| SA
    ESO -->|4| PIA
    PIA -->|5| PIAS
    PIAS -->|6| IAMRole
    IAMRole -->|7| PIA
    PIA -->|8| SecretASM
    SecretASM -->|9| PIA
    PIA -->|10| ESO
    ESO -->|11| K8sSecret
    AppPod -->|12| K8sSecret
    
    %% Styling
    classDef awsService fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#FFFFFF
    classDef k8sResource fill:#326CE5,stroke:#FFFFFF,stroke-width:2px,color:#FFFFFF
    classDef secretStore fill:#4CAF50,stroke:#FFFFFF,stroke-width:2px,color:#FFFFFF
    classDef dataFlow fill:#FFC107,stroke:#FF6F00,stroke-width:2px,color:#000000
    
    class PIAS,IAMRole,IAMPolicy,SecretASM awsService
    class SA,ESO,CSS,ES,K8sSecret,AppPod,PIA k8sResource
    class CSS,ES secretStore
```

## Flow Steps Explanation

1. **ESO Controller** reads ExternalSecret resource
2. **ExternalSecret** references ClusterSecretStore
3. **ClusterSecretStore** specifies ServiceAccount for authentication
4. **ESO** makes AWS API call (intercepted by Pod Identity Agent)
5. **Pod Identity Agent** checks Pod Identity Association
6. **Pod Identity Association** assumes the mapped IAM Role
7. **IAM Role** returns temporary credentials to Pod Identity Agent
8. **Pod Identity Agent** forwards authenticated call to AWS Secrets Manager
9. **AWS Secrets Manager** returns secret data to Pod Identity Agent
10. **Pod Identity Agent** forwards secret data back to ESO Controller
11. **ESO Controller** creates/updates Kubernetes Secret
12. **Application Pod** consumes the Kubernetes Secret

## Prerequisites

- EKS cluster with Pod Identity add-on installed
- AWS CLI configured with appropriate permissions
- kubectl configured to access your EKS cluster
- External Secrets Operator installed

## Step-by-Step Setup

### Step 1: Enable Pod Identity Add-on (if not already enabled)

```bash
aws eks create-addon \
    --cluster-name your-cluster-name \
    --addon-name eks-pod-identity-agent \
    --resolve-conflicts OVERWRITE
```

### Step 2: Create IAM Role and Policy

Create an IAM policy for Secrets Manager access:

```bash
cat <<EOF > secrets-manager-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name ExternalSecretsPolicy \
    --policy-document file://secrets-manager-policy.json
```

Create an IAM role:

```bash
cat <<EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
EOF

aws iam create-role \
    --role-name ExternalSecretsRole \
    --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
    --role-name ExternalSecretsRole \
    --policy-arn arn:aws:iam::ACCOUNT_ID:policy/ExternalSecretsPolicy
```

### Step 3: Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets \
    --create-namespace \
    --set serviceAccount.name=external-secrets
```

### Step 4: Create Pod Identity Association

```bash
aws eks create-pod-identity-association \
    --cluster-name your-cluster-name \
    --namespace external-secrets \
    --service-account external-secrets \
    --role-arn arn:aws:iam::ACCOUNT_ID:role/ExternalSecretsRole
```

### Step 5: Verify Pod Identity Association

```bash
aws eks list-pod-identity-associations --cluster-name your-cluster-name

# Get detailed information
aws eks describe-pod-identity-association \
    --cluster-name your-cluster-name \
    --association-id YOUR_ASSOCIATION_ID
```

### Step 6: Create ClusterSecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2  # Change to your AWS region
      auth:
        serviceAccount:
          name: external-secrets
          namespace: external-secrets
```

Apply the configuration:

```bash
kubectl apply -f cluster-secret-store.yaml
```

### Step 7: Create ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: your-app-namespace
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-credentials
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: your-secret-name
      property: password
  - secretKey: username
    remoteRef:
      key: your-secret-name  
      property: username
```

Apply the configuration:

```bash
kubectl apply -f external-secret.yaml
```

## Verification and Troubleshooting

### Verify Pod Identity Agent

```bash
kubectl get pods -n kube-system -l app=eks-pod-identity-agent
```

### Check ClusterSecretStore Status

```bash
kubectl get clustersecretstore
kubectl describe clustersecretstore aws-secrets-manager
```

### Check ExternalSecret Status

```bash
kubectl get externalsecret app-secret -n your-app-namespace
kubectl describe externalsecret app-secret -n your-app-namespace
```

### Check Generated Kubernetes Secret

```bash
kubectl get secret app-credentials -n your-app-namespace
kubectl describe secret app-credentials -n your-app-namespace
```

### Check ESO Controller Logs

```bash
kubectl logs -n external-secrets deployment/external-secrets -f
```

### Check Pod Identity Agent Logs

```bash
kubectl logs -n kube-system -l app=eks-pod-identity-agent
```

## Common Issues and Solutions

### Issue: ClusterSecretStore shows "SecretStore validation failed"

**Solution**: Ensure the ClusterSecretStore has the correct ServiceAccount reference:

```yaml
spec:
  provider:
    aws:
      auth:
        serviceAccount:
          name: external-secrets
          namespace: external-secrets
```

### Issue: "Access denied" errors

**Solution**: Verify IAM role permissions and Pod Identity Association:

```bash
# Check IAM role policies
aws iam list-attached-role-policies --role-name ExternalSecretsRole

# Check Pod Identity Association
aws eks describe-pod-identity-association \
    --cluster-name your-cluster-name \
    --association-id YOUR_ASSOCIATION_ID
```

### Issue: ESO pods not using the correct ServiceAccount

**Solution**: Verify ESO deployment ServiceAccount:

```bash
kubectl get deployment external-secrets -n external-secrets -o yaml | grep serviceAccount
```

## Benefits of Pod Identity over IRSA

- **Simplified Configuration**: No OIDC provider setup required
- **Better Performance**: Credentials cached by Pod Identity Agent
- **Enhanced Security**: Credentials never stored in pods
- **Improved Observability**: Better logging and audit trails
- **Centralized Management**: Pod Identity Associations managed through AWS APIs

## Security Best Practices

1. **Least Privilege**: Grant only necessary permissions to the IAM role
2. **Resource Restrictions**: Limit Secrets Manager access to specific secrets using resource ARNs
3. **Regular Rotation**: Implement secret rotation policies in AWS Secrets Manager
4. **Monitoring**: Set up CloudTrail logging for secret access auditing
5. **Network Security**: Use VPC endpoints for Secrets Manager if needed

## Next Steps

- Set up secret rotation policies in AWS Secrets Manager
- Implement monitoring and alerting for secret access
- Consider using AWS Config for compliance monitoring
- Set up backup and disaster recovery for critical secrets