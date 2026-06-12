```markdown
# Project Overview

Centralized logging solution for Amazon EKS using Fluent Bit and AWS CloudWatch Logs.

# Components

- EKS Cluster
- Sample Application
- Fluent Bit DaemonSet
- CloudWatch Logs
- IAM Roles for Service Accounts (IRSA)

### Final Architecture

```ascii
                     +-------------------------+
                     |   Sample Application    |
                     |  Namespace: devops-assignment |
                     +------------+------------+
                                  |
                                  | stdout/stderr
                                  v
                     +-------------------------+
                     |  Fluent Bit DaemonSet   |
                     | Namespace: amazon-cloudwatch |
                     +------------+------------+
                                  |
                                  | CloudWatch Logs API
                                  v
                     +-------------------------+
                     |     AWS CloudWatch      |
                     |   Log Group:            |
                     | /eks/devops-assignment/logs |
                     +-------------------------+

```

# Create EKS Cluster

Create cluster:

```bash
eksctl create cluster \
  --name test-cluster \
  --region ap-south-1 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 2
```

This takes ~15–20 minutes.

**Verify:**

```bash
kubectl get nodes
```

# Create Namespace

Create file: `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-assignment
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

# Create CloudWatch Log Group

```bash
aws logs create-log-group \
  --log-group-name /eks/devops-assignment/logs
```

**Verify:**

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /eks/devops-assignment/logs
```

# Create IAM Policy for Fluent Bit

Create file: `cloudwatch-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "logs:CreateLogStream",
        "logs:CreateLogGroup",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

Create policy:

```bash
aws iam create-policy \
  --policy-name FluentBitCloudWatchPolicy \
  --policy-document file://cloudwatch-policy.json
```

# Enable IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster test-cluster \
  --approve
```

# Create IAM role using web identity and attach the above policy and update the trusted policy

After creating the role by attaching the `FluentBitCloudWatchPolicy`, update the trusted relationships as below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::948735414805:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/DBAB5D91573ED2D8377BCF23EB80BE3C"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-south-1.amazonaws.com/id/DBAB5D91573ED2D8377BCF23EB80BE3C:aud": "sts.amazonaws.com",
                    "oidc.eks.ap-south-1.amazonaws.com/id/DBAB5D91573ED2D8377BCF23EB80BE3C:sub": "system:serviceaccount:amazon-cloudwatch:fluent-bit"
                }
            }
        }
    ]
}
```

# Deploy Fluent Bit

## Create ConfigMap

Create file: `fluent-bit-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: amazon-cloudwatch
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Daemon        Off
        Log_Level     info
        Parsers_File  parsers.conf

    # ── INPUT ─────────────────────────────────────────────────────────────
    [INPUT]
        Name              tail
        Tag               kube.<namespace_name>.<pod_name>.<container_name>.<container_id>
        Tag_Regex         (?<pod_name>[a-z0-9](?:[-a-z0-9]*[a-z0-9])?(?:\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-(?<container_id>[a-z0-9]{64})\.log$
        Path              /var/log/containers/*.log
        Parser            cri
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    # ── FILTER 1: Extract namespace/pod/container from Tag via Lua ────────
    # This works even when Kubernetes API server is unreachable.
    # Tag format: kube.<namespace_name>.<pod_name>.<container_name>.<container_id>
    [FILTER]
        Name    lua
        Match   kube.*
        script  extract_k8s_fields.lua
        call    extract_k8s_fields

    # ── FILTER 2: Kubernetes API enrichment (best-effort) ────────────────
    # Still attempt API enrichment. If unreachable, fields from Lua remain.
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.
        Regex_Parser        custom-tag
        Merge_Log           On
        Keep_Log            On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
        Labels              Off
        Annotations         Off

    # ── OUTPUT ────────────────────────────────────────────────────────────
    [OUTPUT]
        Name              cloudwatch_logs
        Match             kube.*
        region            ap-south-1
        log_group_name    /eks/devops-assignment/logs
        log_stream_prefix fluent-bit-
        auto_create_group true
        log_format        json

  # ── Lua script: parse Tag → namespace_name, pod_name, container_name ──
  extract_k8s_fields.lua: |
    function extract_k8s_fields(tag, timestamp, record)
      -- Tag format: kube.<namespace_name>.<pod_name>.<container_name>.<container_id>
      local namespace_name, pod_name, container_name, container_id =
        tag:match("^kube%.([^%.]+)%.([^%.]+)%.([^%.]+)%.(%w+)$")

      if namespace_name then
        record["namespace_name"] = namespace_name
        record["pod_name"]       = pod_name
        record["container_name"] = container_name
      else
        -- Fallback: try splitting on dots generically
        local parts = {}
        for part in tag:gmatch("[^%.]+") do
          table.insert(parts, part)
        end
        -- parts[1]=kube, [2]=namespace, [3]=pod, [4]=container, [5]=container_id
        if #parts >= 4 then
          record["namespace_name"] = parts[2]
          record["pod_name"]       = parts[3]
          record["container_name"] = parts[4]
        end
      end

      return 1, timestamp, record
    end

  parsers.conf: |
    # CRI parser — for containerd runtime (EKS 1.24+ default)
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    # custom-tag parser — used by kubernetes filter Regex_Parser
    [PARSER]
        Name    custom-tag
        Format  regex
        Regex   ^(?<namespace_name>[^.]+)\.(?<pod_name>[a-z0-9](?:[-a-z0-9]*[a-z0-9])?(?:\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)\.(?<container_name>.+)\.(?<container_id>[a-z0-9]{64})$
```

Apply:

```bash
kubectl apply -f fluent-bit-configmap.yaml
```

# Fluent Bit DaemonSet

Create file: `fluent-bit-daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit
  template:
    metadata:
      labels:
        k8s-app: fluent-bit
    spec:
      serviceAccountName: fluent-bit

      containers:
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:latest

        volumeMounts:
        - name: varlog
          mountPath: /var/log

        - name: config
          mountPath: /fluent-bit/etc/

      terminationGracePeriodSeconds: 10

      volumes:
      - name: varlog
        hostPath:
          path: /var/log

      - name: config
        configMap:
          name: fluent-bit-config
```

Apply:

```bash
kubectl apply -f fluent-bit-daemonset.yaml
```

# Create amazon-cloudwatch Namespace

```bash
kubectl create namespace amazon-cloudwatch
```

# Deploy Sample Application

Create file: `sample-app.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: devops-assignment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-container
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "$(date) INFO Application running successfully";
            echo "$(date) ERROR Database connection failed";
            sleep 10;
          done
```

Apply:

```bash
kubectl apply -f sample-app.yaml
```

# Verify Application Logs

Get app pods:

```bash
kubectl get pods -n devops-assignment
```

Check logs:

```bash
kubectl logs <app-pod> -n devops-assignment
```

**Expected output:**

```
INFO Application running successfully
ERROR Database connection failed
```

# Verify CloudWatch Logs

1. Go to **AWS Console** → **CloudWatch** → **Logs** → **Log Groups**
2. Open: `/eks/devops-assignment/logs`
3. Go to **CloudWatch** → **Logs Insights**
4. Run query:

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
```
```