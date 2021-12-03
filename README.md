## Create EKS cluster - ekscluster-sbpath
```
cat eksworkshop.yaml 
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ekscluster-sbpath
  region: us-east-1
  version: "1.21"

availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true
```
eksctl create cluster -f eksworkshop.yaml 

## Create Fargate profile
eksctl create fargateprofile --cluster ekscluster-sbpath --name ns-fargate --namespace ns-fargate

eksctl get fargateprofile --cluster ekscluster-sbpath -o yaml

eksctl utils associate-iam-oidc-provider --cluster ekscluster-sbpath  --approve

## Create service account for x-ray

eksctl create iamserviceaccount --name xray-daemon --namespace ns-fargate --cluster ekscluster-sbpath --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --approve --override-existing-serviceaccounts

kubectl label serviceaccount xray-daemon app=xray-daemon --namespace ns-fargate

## Create Role/ConfigMap required for x-ray
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

## Clone sample project

git clone https://github.com/aws-samples/eks-workshop.git

## Create ECR repo for sample-front and sample-back

aws ecr create-repository --repository-name=sample-front
aws ecr create-repository --repository-name=sample-back


## Configure Frontend app to use x-ray as sidecar
cd eks-workshop/content/intermediate/245_x-ray/sample-front.files

vi main.go
 - remove DaemonAddr:     "xray-service.default:2000",
 - change namespace from default to ns-fargate (from x-ray-sample-back-k8s.default.svc.cluster.local to x-ray-sample-back-k8s.ns-fargate.svc.cluster.local)

vi build.sh
 - Change REPOSITORY to new path of backend

./build.sh

## Deploy frontend application
```
$ cat x-ray-sample-front-k8s.yml 
apiVersion: v1
kind: Service
metadata:
  name: x-ray-sample-front-k8s
  namespace: ns-fargate
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: x-ray-sample-front-k8s
    tier: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: x-ray-sample-front-k8s
  namespace: ns-fargate
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
  selector:
    matchLabels:
      app: x-ray-sample-front-k8s
      tier: frontend
  template:
    metadata:
      labels:
        app: x-ray-sample-front-k8s
        tier: frontend
    spec:
      serviceAccountName: xray-daemon
      containers:
        - name: x-ray-sample-front-k8s
          image: 692050956348.dkr.ecr.us-east-1.amazonaws.com/sample-front:427b2f9879c2e0c9dc49164568ad2d719902dfd0
          securityContext:
            privileged: false
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          ports:
            - containerPort: 8080
        - name: xray-daemon
          image: amazon/aws-xray-daemon
          imagePullPolicy: Always
          command: [ "/usr/bin/xray", "-c", "/aws/xray/config.yaml" ]
          resources:
            limits:
              memory: 24Mi
          ports:
          - name: xray-ingest
            containerPort: 2000
            protocol: UDP
          volumeMounts:
          - name: config-volume
            mountPath: /aws/xray
            readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: xray-config
```

kubectl apply -f x-ray-sample-front-k8s.yml


## Configure Backend app to use x-ray as sidecar
cd eks-workshop/content/intermediate/245_x-ray/sample-back.files

vi main.go
 - remove DaemonAddr:     "xray-service.default:2000",

vi build.sh
 - Change REPOSITORY to new ECR path of backend

./build.sh


## Deploy Backend application
```
$ cat sample-back.files/x-ray-sample-back-k8s.yml 
apiVersion: v1
kind: Service
metadata:
  name: x-ray-sample-back-k8s
  namespace: ns-fargate
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: x-ray-sample-back-k8s
    tier: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: x-ray-sample-back-k8s
  namespace: ns-fargate
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
  selector:
    matchLabels:
      app: x-ray-sample-back-k8s
      tier: backend
  template:
    metadata:
      labels:
        app: x-ray-sample-back-k8s
        tier: backend
    spec:
      serviceAccountName: xray-daemon
      containers:
        - name: x-ray-sample-back-k8s
          image: 692050956348.dkr.ecr.us-east-1.amazonaws.com/sample-back:427b2f9879c2e0c9dc49164568ad2d719902dfd0
          securityContext:
            privileged: false
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          ports:
            - containerPort: 8080
        - name: xray-daemon
          image: amazon/aws-xray-daemon
          imagePullPolicy: Always
          command: [ "/usr/bin/xray", "-c", "/aws/xray/config.yaml" ]
          resources:
            limits:
              memory: 24Mi
          ports:
          - name: xray-ingest
            containerPort: 2000
            protocol: UDP
          volumeMounts:
          - name: config-volume
            mountPath: /aws/xray
            readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: xray-config
```

kubectl apply -f x-ray-sample-back-k8s.yml


## Finally open x-ray console to view traces
