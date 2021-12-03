# Create EKS cluster - ekscluster-sbpath

# Create Fargate profile
eksctl create fargateprofile --cluster ekscluster-sbpath --name ns-fargate --namespace ns-fargate

eksctl get fargateprofile --cluster ekscluster-sbpath -o yaml

eksctl utils associate-iam-oidc-provider --cluster ekscluster-sbpath  --approve

# Create service account for x-ray

eksctl create iamserviceaccount --name xray-daemon --namespace ns-fargate --cluster ekscluster-sbpath --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --approve --override-existing-serviceaccounts

kubectl label serviceaccount xray-daemon app=xray-daemon --namespace ns-fargate

# Create Role/ConfigMap required for x-ray
```
$ cat xray-k8-basics.yaml 
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: xray-daemon
  labels:
    app: xray-daemon
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: xray-daemon
  namespace: ns-fargate
---
# Configuration for AWS X-Ray daemon
apiVersion: v1
kind: ConfigMap
metadata:
  name: xray-config
  namespace: ns-fargate
data:
  config.yaml: |-
    # Maximum buffer size in MB (minimum 3). Choose 0 to use 1% of host memory.
    TotalBufferSizeMB: 0
    # Maximum number of concurrent calls to AWS X-Ray to upload segment documents.
    Concurrency: 8
    # Send segments to AWS X-Ray service in a specific region
    Region: ""
    # Change the X-Ray service endpoint to which the daemon sends segment documents.
    Endpoint: ""
    Socket:
      # Change the address and port on which the daemon listens for UDP packets containing segment documents.
      # Make sure we listen on all IP's by default for the k8s setup
      UDPAddress: 0.0.0.0:2000
    Logging:
      LogRotation: true
      # Change the log level, from most verbose to least: dev, debug, info, warn, error, prod (default).
      LogLevel: prod
      # Output logs to the specified file path.
      LogPath: ""
    # Turn on local mode to skip EC2 instance metadata check.
    LocalMode: false
    # Amazon Resource Name (ARN) of the AWS resource running the daemon.
    ResourceARN: ""
    # Assume an IAM role to upload segments to a different account.
    RoleARN: ""
    # Disable TLS certificate verification.
    NoVerifySSL: false
    # Upload segments to AWS X-Ray through a proxy.
    ProxyAddress: ""
    # Daemon configuration file format version.
    Version: 1
$
```
kubectl  apply  -x xray-k8-basics.yaml 
