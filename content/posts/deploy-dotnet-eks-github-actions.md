---
title: ".NET to EKS with GitHub Actions: OIDC, Helm, canary"
date: 2025-08-28
description: "Secure CI/CD from GitHub to AWS: build .NET, push to ECR, deploy to EKS with Helm, wire RDS, and keep rollouts safe."
tags: [".NET", "EKS", "GitHub Actions", "Helm", "RDS", "S3", "CI/CD", "AWS"]
---

Goal
- Ship a .NET service to EKS with a clean pipeline:
  - GitHub Actions uses OIDC (no long‑lived AWS keys)
  - Builds and pushes image to ECR
  - Helm deploy with safe rolling updates (optional canary)
  - Connect to RDS with TLS
  - Optional: upload static assets to S3

Repo layout
```
.  
├── src/MyApi/. 
│   ├── Program.cs 
│   ├── MyApi.csproj 
│   └── Dockerfile 
├── charts/api/            # simple Helm chart for the API 
│   ├── Chart.yaml 
│   ├── values.yaml 
│   └── templates/ 
│       ├── deployment.yaml 
│       ├── service.yaml  
│       ├── hpa.yaml  
│       └── ingress.yaml 
└── .github/workflows/deploy.yml 
```

Dockerfile (multi-stage, non‑root)
```dockerfile
# src/MyApi/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
# run as non-root
RUN adduser --disabled-password --gecos "" app && chown -R app:app /app
USER app
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Minimal Program.cs (health + ready)
```
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapGet("/ping", () => "pong");
app.MapHealthChecks("/healthz");     // liveness
app.MapHealthChecks("/readyz");      // readiness (wire app dependencies here)

app.Run();
```

Helm chart: values.yaml
```
image:
  repository: <aws_account_id>.dkr.ecr.<region>.amazonaws.com/myapi
  tag: "latest"
  pullPolicy: IfNotPresent

replicaCount: 3

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

env:
  - name: ASPNETCORE_ENVIRONMENT
    value: "Production"
  - name: ConnectionStrings__AppDb
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: connectionString

livenessProbe:
  path: /healthz
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  path: /readyz
  initialDelaySeconds: 10
  periodSeconds: 10

ingress:
  enabled: false

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

Helm chart: templates/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
  labels: {{- include "api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: {{- include "api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "api.selectorLabels" . | nindent 8 }}
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env: {{- toYaml .Values.env | nindent 12 }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet: { path: {{ .Values.livenessProbe.path | quote }}, port: {{ .Values.service.targetPort }} }
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
          readinessProbe:
            httpGet: { path: {{ .Values.readinessProbe.path | quote }}, port: {{ .Values.service.targetPort }} }
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
```

Helm chart: templates/service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "api.fullname" . }}
  labels: {{- include "api.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector: {{- include "api.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

Helm chart: templates/hpa.yaml
```
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "api.fullname" . }}
spec:
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.targetCPUUtilizationPercentage }}
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "api.fullname" . }}
{{- end }}
```

Helm helpers (templates/_helpers.tpl)
```
{{- define "api.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "api.fullname" -}}
{{- printf "%s-%s" .Release.Name (include "api.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "api.labels" -}}
app.kubernetes.io/name: {{ include "api.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: Helm
{{- end -}}

{{- define "api.selectorLabels" -}}
app.kubernetes.io/name: {{ include "api.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

RDS: connection string secret (example)
```
# create a k8s Secret with TLS enforced; prefer Secrets Manager + External Secrets in prod
kubectl -n prod create secret generic app-secrets \
  --from-literal=connectionString="Server=<rds-endpoint>;Database=<db>;User Id=<user>;Password=<pass>;Encrypt=True;TrustServerCertificate=False;SslMode=Require"
```

GitHub → AWS OIDC role (trust policy)
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::<account_id>:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:yourgithub/yourrepo:*"
        }
      }
    }
  ]
}
```

Role policy (push to ECR + deploy to EKS)
```
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["ecr:GetAuthorizationToken"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["ecr:BatchCheckLayerAvailability","ecr:CompleteLayerUpload","ecr:UploadLayerPart","ecr:InitiateLayerUpload","ecr:PutImage","ecr:BatchGetImage","ecr:DescribeRepositories","ecr:CreateRepository"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["eks:DescribeCluster"], "Resource": "arn:aws:eks:<region>:<account_id>:cluster/<cluster_name>" },
    { "Effect": "Allow", "Action": ["ssm:GetParameter","secretsmanager:GetSecretValue"], "Resource": "*" }
  ]
}
```

Map CI role to EKS (so kubectl/Helm can act)
```
# one-time, from admin machine
eksctl create iamidentitymapping \
  --cluster <cluster_name> \
  --region <region> \
  --arn arn:aws:iam::<account_id>:role/<role_name_for_ci> \
  --group system:masters \
  --username github-ci
```

GitHub Actions workflow (.github/workflows/deploy.yml)
```
name: ci-cd
on:
  push:
    branches: [ main ]
  workflow_dispatch: {}

env:
  AWS_REGION: ap-southeast-1
  ECR_REPO: myapi
  CLUSTER_NAME: prod-eks
  NAMESPACE: prod
  RELEASE: prod
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Run tests
        run: dotnet test --nologo --verbosity minimal

      - name: Configure AWS (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account_id>:role/<role_name_for_ci>
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        working-directory: src/MyApi
        run: |
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          IMAGE_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
          docker build -t ${IMAGE_URI} -f Dockerfile .
          docker push ${IMAGE_URI}
          echo "IMAGE_URI=${IMAGE_URI}" >> $GITHUB_ENV

      - name: Setup kubectl and helm
        run: |
          curl -sSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}

      - name: Create namespace if missing
        run: kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}

      - name: Set image tag in values
        run: |
          sed -i "s|tag: .*|tag: \"${IMAGE_TAG}\"|g" charts/api/values.yaml
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          sed -i "s|<aws_account_id>.dkr.ecr.<region>.amazonaws.com|${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com|g" charts/api/values.yaml

      - name: Deploy with Helm (rolling update)
        run: |
          helm upgrade --install ${RELEASE} charts/api \
            --namespace ${NAMESPACE} \
            --wait --timeout 5m

      - name: Post-deploy smoke
        run: |
          kubectl -n ${NAMESPACE} rollout status deploy/${RELEASE}-api --timeout=120s
```

Optional: manual canary (10% traffic)

Create a second deployment with replicaCount=1 named prod-api-canary and route via header/path in your Ingress (ALB supports header‑based rules). Keep it simple if you don’t have an Ingress yet—stick to rolling updates with maxUnavailable=0, maxSurge=1.
S3 (optional) — upload static assets in CI

```
      - name: Sync static to S3
        if: false  # flip to true when you have /static to publish
        run: |
          aws s3 sync static/ s3://myapi-static-bucket/ --delete
```

When to use

You want a clean path from git push to production with no shared AWS keys.
You run EKS and deploy containerized .NET services that talk to RDS.
You prefer Helm for release tracking and rollbacks.
Failure modes and fast fixes

OIDC “AccessDenied”: check the role trust policy repo filter (sub must match repo:owner/name:*).
kubectl forbidden: map the role to cluster (aws-auth) and retry; avoid mapping wildcards.
Pods crashloop on startup: check ConnectionStrings__AppDb, TLS params, and security groups to RDS.
Slow rollouts or timeouts: set readiness to only pass when downstreams are reachable; keep maxUnavailable=0 for zero downtime; use helm rollback if needed:
helm rollback prod 1 --namespace prod

Checklist
 ECR repo exists
 OIDC role created, trust policy matches repo
 Role mapped to EKS cluster
 Namespace + Secret with DB connection string
 Helm chart installed with probes, HPA, PDB
 CI builds, pushes, deploys, and waits for rollout

 