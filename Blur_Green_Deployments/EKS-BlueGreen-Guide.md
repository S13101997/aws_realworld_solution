# Complete EKS Blue-Green Deployment Implementation with Argo Rollouts

## Overview

EKS blue-green deployments use **Argo Rollouts + AWS Load Balancer Controller (ALB Ingress)** to achieve Kubernetes-native, zero-downtime updates. Traffic is gradually shifted between blue and green ReplicaSets while health and metrics are monitored.

---

## 1. Architecture Overview

High-level flow:

- Users → ALB → Ingress → **Active Service** → Blue/Green pods (via Argo Rollout)
- Argo Rollouts manages:
  - Blue (current active) ReplicaSet
  - Green (new version) ReplicaSet
  - Promotion, traffic switching, and rollback

Main components:

- **EKS cluster** with worker nodes (optionally GPU nodes)
- **AWS Load Balancer Controller** to create/manage ALBs from Ingress
- **Argo Rollouts** to implement blue-green deployment logic
- **Active Service** (live traffic) and **Preview Service** (for testing green)
- **Ingress** pointing at Active Service

---

## 2. Prerequisites and Installation

### 2.1 Create EKS Cluster

```bash
eksctl create cluster \
  --name production \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed
```

(If you need GPUs, use `g5.xlarge`/similar instead of `m5.large`.)

### 2.2 Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=production \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system
```

(You also need the recommended IAM role for the controller, via IAM Roles for Service Accounts.)

### 2.3 Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Optional: install the kubectl plugin for Rollouts:

```bash
brew install argoproj/tap/kubectl-argo-rollouts  # macOS / Homebrew

# Or download from GitHub releases
```

---

## 3. Core Kubernetes Manifests

Below is a minimal but realistic example using Argo Rollouts' blue-green strategy.

### 3.1 Rollout (Blue-Green Strategy)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: my-app
  strategy:
    blueGreen:
      activeService: app-active      # Service receiving production traffic
      previewService: app-preview    # Service for testing new version
      previewReplicaCount: 2
      autoPromotionEnabled: true
      autoPromotionSeconds: 300      # Wait 5 min before auto promotion
      scaleDownDelaySeconds: 120     # Keep old pods around briefly
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

### 3.2 Active and Preview Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-active
  namespace: production
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: app-preview
  namespace: production
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### 3.3 Ingress with AWS Load Balancer Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-active
                port:
                  name: http
```

The **Ingress** always points at `app-active`. Argo Rollouts changes which pods are behind `app-active` by switching the active ReplicaSet during promotion.

---

## 4. Step-by-Step Blue-Green Workflow

### 4.1 Deploy Initial (Blue) Version

```bash
kubectl create namespace production

kubectl apply -f rollout.yaml      # contains Rollout + Services + Ingress
```

Check rollout status:

```bash
kubectl argo rollouts get rollout app-rollout -n production
kubectl get pods -n production -l app=my-app
```

At this point:

- `v1.0` pods (Blue) are running
- `app-active` points to Blue ReplicaSet
- ALB forwards all production traffic to `app-active`

### 4.2 Deploy New (Green) Version

Update the image in the Rollout spec to `v2.0` and apply:

```bash
kubectl set image rollout/app-rollout \
  app=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v2.0 \
  -n production
```

Argo Rollouts will:

1. Create a new ReplicaSet (Green) with `v2.0`
2. Bring up `previewReplicaCount` pods behind `app-preview`
3. Keep production traffic on Blue (`app-active`) while Green is validated
4. After `autoPromotionSeconds` (300s) and if no analysis step fails, it will:
   - Promote Green to active
   - Re-route `app-active` to Green ReplicaSet
   - Scale down Blue after `scaleDownDelaySeconds`

Watch it in real time:

```bash
kubectl argo rollouts get rollout app-rollout -n production -w
```

---

## 5. Manual Control (Promote/Roll Back)

You can override automatic promotion:

### 5.1 Pause and Manually Promote Green

```bash
# Pause rollout (optional, if you disabled autoPromotion)
kubectl argo rollouts pause app-rollout -n production

# When ready after testing preview:
kubectl argo rollouts promote app-rollout -n production
```

Promotion causes:

- `app-active` → Green ReplicaSet
- Blue pods kept briefly, then scaled down

### 5.2 Roll Back to Previous Blue Version

Argo Rollouts stores previous versions as history:

```bash
# List rollout history
kubectl argo rollouts get rollout app-rollout -n production --revision

# Roll back to previous revision (e.g., revision 1)
kubectl argo rollouts rollback app-rollout --to-revision=1 -n production
```

This will:

- Switch back to the previous ReplicaSet (old image)
- Make it active again (blue) behind `app-active`

---

## 6. Adding Metrics-Based Analysis (Optional but Recommended)

You can integrate Prometheus to gate promotion based on error rate/latency.

Example `AnalysisTemplate`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  metrics:
    - name: http-success-rate
      successCondition: result[0] >= 99
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus-server.production.svc.cluster.local
          query: |
            100 * sum(rate(http_requests_total{app="my-app",status!~"5.."}[5m])) /
            sum(rate(http_requests_total{app="my-app"}[5m]))
```

Then reference it from the Rollout:

```yaml
spec:
  strategy:
    blueGreen:
      ...
      scaleDownDelaySeconds: 120
  analysis:
    templates:
      - templateName: success-rate
    startingStep: 0
```

Now promotion will only proceed if the Prometheus query passes (e.g., success rate ≥ 99%).

---

## 7. Complete Deployment Script

Here's a full bash script to orchestrate the entire deployment:

```bash
#!/bin/bash
# deploy-eks-blue-green.sh

set -e

# Configuration
CLUSTER_NAME="production"
REGION="us-east-1"
NAMESPACE="production"
ROLLOUT_NAME="app-rollout"
IMAGE_REPO="123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app"
NEW_VERSION="v2.0"
OLD_VERSION="v1.0"

echo "=== EKS Blue-Green Deployment Script ==="

# 1. Ensure cluster context
echo "[1/5] Setting kubectl context..."
aws eks update-kubeconfig --name $CLUSTER_NAME --region $REGION

# 2. Build and push Docker image
echo "[2/5] Building and pushing Docker image..."
docker build -t $IMAGE_REPO:$NEW_VERSION .
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $(echo $IMAGE_REPO | cut -d'/' -f1)
docker push $IMAGE_REPO:$NEW_VERSION

# 3. Create namespace if needed
echo "[3/5] Creating namespace..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# 4. Apply Rollout manifests
echo "[4/5] Applying Rollout manifests..."
kubectl apply -f rollout.yaml -n $NAMESPACE

# 5. Trigger deployment
echo "[5/5] Triggering green deployment..."
kubectl set image rollout/$ROLLOUT_NAME \
  app=$IMAGE_REPO:$NEW_VERSION \
  -n $NAMESPACE

# Wait for rollout
echo "Waiting for rollout to complete..."
kubectl argo rollouts get rollout $ROLLOUT_NAME -n $NAMESPACE -w

echo "✅ Deployment complete!"
```

---

## 8. Traffic Shift Timeline

Timeline for automatic blue-green promotion:

| Time | Blue % | Green % | Phase | Status |
|------|--------|---------|-------|--------|
| 0m | 100% | 0% | Blue Live | Initial state |
| 1m | 100% | 0% | Green Provisioning | 2 preview pods created |
| 3m | 100% | 0% | Health Checks | Green pods ready |
| 5m | 100% | 0% | Pre-promotion | Waiting for auto-promotion |
| 5m+ | 0% | 100% | Green Live | Promotion complete |
| 7m | 0% | 100% | Blue Draining | Blue scaled to 0 |

---

## 9. Monitoring and Observability

### 9.1 Watch Rollout Status

```bash
# Real-time rollout status
kubectl argo rollouts get rollout app-rollout -n production -w

# Detailed rollout info
kubectl argo rollouts get rollout app-rollout -n production -o yaml

# Pod status
kubectl get pods -n production -l app=my-app -o wide
```

### 9.2 Check Service Routing

```bash
# View service endpoints
kubectl get endpoints app-active app-preview -n production -o yaml

# Test connectivity
kubectl port-forward svc/app-active 8080:80 -n production &
curl http://localhost:8080/healthz
```

### 9.3 View Logs

```bash
# Follow green pod logs during deployment
kubectl logs -f deployment/app-rollout-green-xxxxx -n production

# Check all pods
kubectl logs -n production -l app=my-app --tail=50 --timestamps=true
```

---

## 10. Rollback Scenarios

### 10.1 Automatic Rollback (Analysis Failed)

If metrics-based analysis fails, Argo automatically rolls back:

```bash
# This happens automatically if:
# - Success rate drops below 99%
# - Latency exceeds threshold
# - Pod crashes

# View rollout status to see failure
kubectl argo rollouts get rollout app-rollout -n production
```

### 10.2 Manual Rollback

```bash
# Roll back to previous version
kubectl argo rollouts rollback app-rollout -n production

# Roll back to specific revision
kubectl argo rollouts rollback app-rollout --to-revision=1 -n production

# Check rollout history
kubectl argo rollouts get rollout app-rollout -n production --revision
```

### 10.3 Emergency Rollback

```bash
# Immediate rollback (abort current deployment)
kubectl argo rollouts abort app-rollout -n production

# Check status
kubectl argo rollouts get rollout app-rollout -n production
```

---

## 11. Advanced Patterns

### 11.1 Canary with Traffic Weight

```yaml
strategy:
  blueGreen:
    activeService: app-active
    previewService: app-preview
    previewReplicaCount: 3
    autoPromotionEnabled: false  # Manual promotion
    scaleDownDelaySeconds: 300
```

### 11.2 Gradual Traffic Shift

```yaml
strategy:
  blueGreen:
    activeService: app-active
    previewService: app-preview
    previewReplicaCount: 6
    autoPromotionEnabled: true
    autoPromotionSeconds: 600    # 10 min canary window
```

### 11.3 Analysis Gate

```yaml
spec:
  strategy:
    blueGreen:
      activeService: app-active
      autoPromotionEnabled: false
  analysis:
    templates:
      - templateName: success-rate
        interval: 1m
        initialDelay: 5m
```

---

## 12. Troubleshooting

### Issue: Rollout Stuck in "Progressing"

```bash
# Check rollout events
kubectl describe rollout app-rollout -n production

# Check if green pods are ready
kubectl get pods -n production -l app=my-app -o wide

# Check service endpoints
kubectl get endpoints app-active app-preview -n production
```

### Issue: Green Pods Crash

```bash
# View logs
kubectl logs -n production -l app=my-app --tail=100

# Check resource constraints
kubectl describe nodes

# Increase resources in Rollout spec
kubectl edit rollout app-rollout -n production
```

### Issue: Traffic Not Routing to Green

```bash
# Verify service selector
kubectl get svc app-active -n production -o yaml | grep -A5 selector

# Check ALB target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...

# Verify ingress
kubectl get ingress app-ingress -n production -o yaml
```

---

## 13. Cost Implications

| Resource | Cost Impact | Notes |
|----------|-------------|-------|
| Running 2 ReplicaSets | 2x pods | Temporary during promotion |
| ALB usage | Minimal | Shared across services |
| Kubernetes API calls | Negligible | Argo Rollouts traffic |
| CloudWatch monitoring | Minimal | ALB metrics |

**Total cost of blue-green during deployment:** ~2x compute for 5-10 minutes

---

## 14. Best Practices

1. **Health Checks**: Always define readiness and liveness probes
2. **Resource Limits**: Set requests/limits to avoid node resource issues
3. **Gradual Promotion**: Use 5-10 min auto-promotion windows
4. **Metrics Analysis**: Integrate Prometheus for automated validation
5. **Manual Testing**: Use preview service for smoke tests before promotion
6. **Monitoring**: Watch pod logs and service endpoints during deployment
7. **Documentation**: Track deployment history and rollback decisions
8. **Rehearse Rollbacks**: Test manual rollback procedures regularly

---

## 15. Production Checklist

- [ ] EKS cluster running with 3+ nodes
- [ ] AWS Load Balancer Controller installed
- [ ] Argo Rollouts installed and working
- [ ] Rollout spec configured with blue-green strategy
- [ ] Active and Preview services created
- [ ] Ingress pointing to app-active
- [ ] Health checks defined for liveness and readiness
- [ ] Docker image builds and pushes to ECR
- [ ] Deployment script tested end-to-end
- [ ] Rollback procedure documented and tested
- [ ] Prometheus integration (optional but recommended)
- [ ] Monitoring and alerting configured
- [ ] Team trained on blue-green deployment process

---

## 16. Complete Example Deployment

Full end-to-end example with all components:

```bash
# 1. Create cluster
eksctl create cluster \
  --name production \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type m5.large \
  --nodes 3

# 2. Install prerequisites
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 3. Deploy initial version
kubectl apply -f rollout.yaml

# 4. Build new version
docker build -t my-app:v2.0 .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v2.0

# 5. Deploy new version
kubectl set image rollout/app-rollout \
  app=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v2.0 \
  -n production

# 6. Monitor deployment
kubectl argo rollouts get rollout app-rollout -n production -w

# 7. Verify success
kubectl argo rollouts get rollout app-rollout -n production
```

---

## Conclusion

EKS blue-green deployments via Argo Rollouts provide:

✓ **Kubernetes-native** deployment strategy  
✓ **Zero-downtime** updates with instant rollback  
✓ **Flexible** traffic management (manual or automatic)  
✓ **Metrics-based** validation and promotion gates  
✓ **GitOps-ready** (integrates with ArgoCD)  
✓ **Production-tested** by enterprises globally  

This approach scales from small apps to enterprise systems handling millions of requests per minute.
