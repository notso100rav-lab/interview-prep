# Kubernetes Basics for Java Developers

> Java Backend Engineer Interview Prep — Chapter 4.3

---

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Deployment & Rolling Updates](#deployment--rolling-updates)
3. [Services](#services)
4. [ConfigMap & Secret](#configmap--secret)
5. [Ingress](#ingress)
6. [HPA (Horizontal Pod Autoscaler)](#hpa)
7. [Probes](#probes)
8. [Resources & Namespaces](#resources--namespaces)
9. [kubectl Cheat Sheet](#kubectl-cheat-sheet)
10. [Full Spring Boot YAML Example](#full-spring-boot-yaml-example)
11. [Q&A](#qa)

---

## Core Concepts

| Resource | Description |
|----------|-------------|
| **Pod** | Smallest deployable unit; one or more containers sharing network/storage |
| **Deployment** | Declarative management of Pods; handles rollouts, rollbacks, scaling |
| **ReplicaSet** | Ensures N pod replicas are running (usually managed by Deployment) |
| **Service** | Stable network endpoint to reach Pods (load balancing, DNS) |
| **ConfigMap** | Non-sensitive configuration data |
| **Secret** | Sensitive data (base64 encoded, not encrypted by default) |
| **Ingress** | HTTP/HTTPS routing rules into the cluster |
| **HPA** | Automatically scales Deployment based on metrics |
| **Namespace** | Virtual cluster for resource isolation |
| **PersistentVolume** | Storage abstraction for stateful workloads |

---

## Deployment & Rolling Updates

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max extra pods during update
      maxUnavailable: 0    # always keep 3 pods running (zero-downtime)
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      terminationGracePeriodSeconds: 60    # time to handle SIGTERM before SIGKILL
      containers:
        - name: myapp
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "768Mi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30      # 30 * 10s = 5min startup window
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]   # drain connections before SIGTERM
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: ["myapp"]
                topologyKey: kubernetes.io/hostname   # spread across nodes
```

### Rollout Commands

```bash
# Deploy
kubectl apply -f deployment.yaml

# Watch rollout progress
kubectl rollout status deployment/myapp -n production

# View rollout history
kubectl rollout history deployment/myapp -n production

# View specific revision
kubectl rollout history deployment/myapp --revision=2 -n production

# Rollback to previous version
kubectl rollout undo deployment/myapp -n production

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=1 -n production

# Pause rollout (canary testing)
kubectl rollout pause deployment/myapp -n production

# Resume rollout
kubectl rollout resume deployment/myapp -n production

# Manual scale
kubectl scale deployment myapp --replicas=5 -n production

# Update image (triggers rolling update)
kubectl set image deployment/myapp myapp=myapp:2.0.0 -n production
```

---

## Services

```yaml
# service.yaml — ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  type: ClusterIP       # default — only accessible within cluster
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80          # service port
      targetPort: 8080  # container port
```

```yaml
# NodePort — accessible via any node IP + nodePort
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # 30000-32767 range
```

```yaml
# LoadBalancer — provisions cloud load balancer (AWS ELB, GCP LB)
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

---

## ConfigMap & Secret

### ConfigMap — Environment Variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  LOGGING_LEVEL_ROOT: "INFO"
  LOGGING_LEVEL_COM_EXAMPLE: "DEBUG"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "myapp"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
```

### ConfigMap — Volume Mount (application.yml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-application-config
  namespace: production
data:
  application-prod.yml: |
    server:
      port: 8080
      shutdown: graceful
    spring:
      datasource:
        url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
        hikari:
          maximum-pool-size: 10
      cache:
        type: redis
    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus,metrics
      endpoint:
        health:
          probes:
            enabled: true
```

```yaml
# Mount ConfigMap as file in deployment
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: app-config
          mountPath: /config
          readOnly: true
      env:
        - name: SPRING_CONFIG_LOCATION
          value: "file:/config/application-prod.yml"
  volumes:
    - name: app-config
      configMap:
        name: myapp-application-config
```

### Secret

```yaml
# Create secret imperatively
kubectl create secret generic myapp-secrets \
  --from-literal=DB_PASSWORD=supersecret \
  --from-literal=JWT_SECRET=jwtsigningkey \
  -n production

# Or declaratively (base64 encode values)
# echo -n 'supersecret' | base64
# c3VwZXJzZWNyZXQ=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=        # base64("supersecret")
  JWT_SECRET: and0c2lnbmluZ2tleQ==      # base64("jwtsigningkey")
  REDIS_PASSWORD: cmVkaXNwYXNz          # base64("redispass")
```

```yaml
# Use with envFrom (all keys become env vars)
spec:
  containers:
    - envFrom:
        - secretRef:
            name: myapp-secrets

# Or individual key reference
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: DB_PASSWORD
```

### External Secrets Operator (Best Practice)

```yaml
# Sync from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /myapp/prod/db-credentials
        property: password
```

---

## Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls-cert    # cert-manager creates this
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
          - path: /actuator/health
            pathType: Exact
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # scale when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80     # scale when avg memory > 80%
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second    # custom metric via KEDA or Prometheus Adapter
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4          # add max 4 pods at once
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1           # remove max 1 pod at once
          periodSeconds: 60
```

---

## Probes

```yaml
# Three probe types for Spring Boot with Actuator
spec:
  containers:
    - name: myapp
      # Startup Probe — checked FIRST, prevents premature liveness checks
      # Gives app up to 5 minutes (30 * 10s) to start
      startupProbe:
        httpGet:
          path: /actuator/health
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      # Liveness Probe — if fails, container is KILLED and restarted
      # Detect deadlock, OOM, hung threads
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 0      # startup probe handles delay
        periodSeconds: 15
        failureThreshold: 3
        timeoutSeconds: 5

      # Readiness Probe — if fails, pod removed from Service endpoints
      # Detect DB connection issues, downstream service unavailable
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 3
        successThreshold: 1
        timeoutSeconds: 5
```

```yaml
# application.yml — enable Kubernetes probes
management:
  endpoint:
    health:
      probes:
        enabled: true      # enables /actuator/health/liveness and /readiness
      show-details: always
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

```java
// Manually transition readiness state (e.g., during warm-up)
@Component
@RequiredArgsConstructor
public class WarmUpService {

    private final ApplicationContext context;

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        AvailabilityChangeEvent.publish(context, ReadinessState.REFUSING_TRAFFIC);
        // do warm-up: load caches, establish connections
        doWarmUp();
        AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
    }
}
```

---

## Resources & Namespaces

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: backend
```

```yaml
# resource limits via LimitRange (default for namespace)
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

```yaml
# ResourceQuota — limit total resources in namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
```

---

## kubectl Cheat Sheet

```bash
# ── Context & Namespace ──────────────────────────
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=production

# ── Apply / Create ───────────────────────────────
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/              # apply all files in directory
kubectl apply -k ./overlays/prod/    # kustomize

# ── Get Resources ────────────────────────────────
kubectl get pods -n production
kubectl get pods -n production -o wide     # with node info
kubectl get all -n production              # pods, services, deployments, etc.
kubectl get pods --watch -n production     # live watch
kubectl get pods -l app=myapp -n production  # label selector

# ── Describe ─────────────────────────────────────
kubectl describe pod myapp-abc123 -n production
kubectl describe deployment myapp -n production
kubectl describe service myapp-service -n production

# ── Logs ─────────────────────────────────────────
kubectl logs myapp-abc123 -n production
kubectl logs -f myapp-abc123 -n production            # follow
kubectl logs myapp-abc123 -c init-container -n production  # specific container
kubectl logs --previous myapp-abc123 -n production    # crashed container logs
kubectl logs -l app=myapp --tail=100 -n production    # all pods with label

# ── Exec / Shell ─────────────────────────────────
kubectl exec -it myapp-abc123 -n production -- /bin/sh
kubectl exec myapp-abc123 -n production -- curl localhost:8080/actuator/health

# ── Port Forward ─────────────────────────────────
kubectl port-forward pod/myapp-abc123 8080:8080 -n production
kubectl port-forward svc/myapp-service 8080:80 -n production
kubectl port-forward deployment/myapp 8080:8080 -n production

# ── Rollout ──────────────────────────────────────
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production
kubectl rollout undo deployment/myapp --to-revision=2 -n production
kubectl rollout restart deployment/myapp -n production   # rolling restart

# ── Scale ────────────────────────────────────────
kubectl scale deployment myapp --replicas=5 -n production
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=70

# ── Delete ───────────────────────────────────────
kubectl delete pod myapp-abc123 -n production         # pod restarts (controlled by Deployment)
kubectl delete deployment myapp -n production
kubectl delete -f deployment.yaml

# ── Debugging ────────────────────────────────────
kubectl get events -n production --sort-by=.lastTimestamp
kubectl top pods -n production                        # CPU/memory usage
kubectl top nodes
kubectl debug pod/myapp-abc123 -it --image=busybox    # ephemeral debug container
```

---

## Full Spring Boot YAML Example

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  DB_HOST: "postgres-service.production.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "myapp"
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=
  JWT_SECRET: and0c2lnbmluZ2tleQ==
---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: myapp
          image: myapp:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "768Mi"
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      name: http
---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [api.myapp.com]
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## Q&A

### Q1 🟢 What is the difference between a Pod and a Deployment in Kubernetes?

<details><summary>Click to reveal answer</summary>

A **Pod** is the smallest deployable unit — one or more containers sharing network and storage. If a Pod dies, it is not restarted automatically.

A **Deployment** manages a set of identical Pods via a **ReplicaSet**. It ensures:
- The desired number of replicas are running
- Rolling updates with zero downtime
- Rollback to previous versions

```bash
# Raw pod (not recommended for production)
kubectl run myapp --image=myapp:latest  # creates a standalone pod

# Deployment (recommended)
kubectl create deployment myapp --image=myapp:latest --replicas=3
```

Always use Deployments (or StatefulSets for stateful apps) in production, never raw Pods.

</details>

---

### Q2 🟢 What are the three types of Kubernetes Services and when do you use each?

<details><summary>Click to reveal answer</summary>

| Type | Access Scope | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal cluster only | Microservice-to-microservice communication |
| **NodePort** | Via node IP + port (30000-32767) | Dev/test external access |
| **LoadBalancer** | External via cloud LB (ELB, GCP LB) | Production external traffic |

```yaml
# ClusterIP (default) — internal only
spec:
  type: ClusterIP   # or omit type

# NodePort — external via any node
spec:
  type: NodePort
  ports:
    - nodePort: 30080

# LoadBalancer — cloud managed
spec:
  type: LoadBalancer
```

In production, use **Ingress** (with ClusterIP service) instead of LoadBalancer — Ingress provides host-based routing, TLS termination, and uses a single cloud LB for all services.

</details>

---

### Q3 🟡 Explain the difference between liveness, readiness, and startup probes.

<details><summary>Click to reveal answer</summary>

| Probe | Failure Action | Purpose |
|-------|---------------|---------|
| **Startup** | Kill + restart container | Slow-starting apps; disables liveness until started |
| **Liveness** | Kill + restart container | Detect deadlocks, OOM, hung threads |
| **Readiness** | Remove from Service endpoints | Detect when app can't serve traffic (DB down, etc.) |

**Execution order**: startup → (liveness + readiness in parallel)

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30   # 30 * 10s = 5 min to start
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

**Key difference**: Liveness restart is disruptive (traffic interrupted). Readiness just removes the pod from load balancing — the pod keeps running.

</details>

---

### Q4 🟡 What is a ConfigMap and how does it differ from a Secret?

<details><summary>Click to reveal answer</summary>

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data (passwords, keys) |
| **Storage** | Plain text in etcd | Base64 encoded in etcd (NOT encrypted by default) |
| **Encryption** | No | Optional (enable etcd encryption at rest) |
| **Size limit** | 1 MiB | 1 MiB |
| **Usage** | `configMapRef`, `configMapKeyRef`, volume | `secretRef`, `secretKeyRef`, volume |

**Important**: Kubernetes Secrets are base64 encoded, NOT encrypted. For true security:
- Enable **etcd encryption at rest** in cluster config
- Use **External Secrets Operator** with AWS Secrets Manager or HashiCorp Vault
- Use **Sealed Secrets** (encrypt with cluster public key, safe to commit)

</details>

---

### Q5 🔴 How does a rolling update work in Kubernetes and how do you configure zero-downtime deploys?

<details><summary>Click to reveal answer</summary>

Rolling update replaces pods one at a time:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1         # create 1 new pod before removing old
    maxUnavailable: 0   # never reduce below desired replicas
```

**Zero-downtime checklist:**
1. `maxUnavailable: 0` — never kill old pod until new is ready
2. **Readiness probe** — new pod only joins Service when ready
3. **preStop hook** — sleep to let load balancer drain connections
4. `terminationGracePeriodSeconds` — time for graceful shutdown
5. Spring Boot `server.shutdown: graceful` — finish in-flight requests

```yaml
containers:
  - lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]  # drain LB before SIGTERM
terminationGracePeriodSeconds: 60
```

**Sequence**: create new pod → readiness passes → added to Service → preStop hook → SIGTERM → graceful shutdown → pod removed.

</details>

---

### Q6 🟡 How does HPA (Horizontal Pod Autoscaler) work?

<details><summary>Click to reveal answer</summary>

HPA queries **metrics-server** (or Prometheus Adapter for custom metrics) every 15s and adjusts replica count:

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

Example: 3 replicas at 90% CPU, target 70%:
```
desiredReplicas = ceil[3 * (90 / 70)] = ceil[3.86] = 4
```

**Requirements:**
- `metrics-server` must be installed in cluster
- Pod `resources.requests` must be set (HPA uses requests as baseline)

```bash
# Check HPA status
kubectl get hpa -n production
kubectl describe hpa myapp-hpa -n production

# Output:
# NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
# myapp-hpa  Deployment/myapp      45%/70%   2         20        3
```

**Stabilization window** prevents flapping:
- Scale up: 60s (quick response to load)
- Scale down: 300s (wait 5 min before reducing)

</details>

---

### Q7 🟢 What is the difference between `kubectl apply` and `kubectl create`?

<details><summary>Click to reveal answer</summary>

| Command | Behavior |
|---------|----------|
| `kubectl create` | Creates resource; fails if already exists |
| `kubectl apply` | Creates or updates resource; idempotent |
| `kubectl replace` | Replaces entire resource; fails if not exists |

```bash
kubectl create -f deployment.yaml   # fails if deployment already exists
kubectl apply -f deployment.yaml    # create or update (idempotent — use in CI/CD)
kubectl apply -f ./k8s/             # apply all YAML files in directory
```

`apply` stores the last-applied configuration as an annotation (`kubectl.kubernetes.io/last-applied-configuration`) to compute a three-way merge on next apply.

</details>

---

### Q8 🔴 How do you debug a CrashLoopBackOff pod?

<details><summary>Click to reveal answer</summary>

`CrashLoopBackOff` means the container starts, crashes, and Kubernetes keeps restarting it with exponential backoff.

```bash
# Step 1: Check pod status and events
kubectl get pod myapp-abc123 -n production
kubectl describe pod myapp-abc123 -n production
# Look at: Events section, Last State, Exit Code

# Step 2: View logs of crashed container
kubectl logs myapp-abc123 -n production --previous
# --previous shows logs from BEFORE the crash

# Step 3: Common exit codes
# Exit 1: App exception (check app logs)
# Exit 137: OOMKilled (increase memory limits)
# Exit 143: SIGTERM (check shutdown hooks)

# Step 4: Check resource limits
kubectl describe pod myapp-abc123 | grep -A5 "Limits\|Requests"

# Step 5: Debug interactively (if container has shell)
kubectl debug pod/myapp-abc123 -it --image=busybox --copy-to=myapp-debug

# Step 6: Check configmap/secret mounts
kubectl exec myapp-abc123 -- env | grep -i db
kubectl exec myapp-abc123 -- cat /config/application.yml
```

Common causes: wrong image tag, missing environment variables, failed DB connection on startup, OOMKilled, wrong command/entrypoint.

</details>

---

### Q9 🟡 What is pod anti-affinity and why is it important?

<details><summary>Click to reveal answer</summary>

Pod anti-affinity prevents multiple pods of the same application from being scheduled on the same node. This provides **high availability** — if a node fails, not all pods die.

```yaml
spec:
  affinity:
    podAntiAffinity:
      # Hard rule — NEVER schedule on same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["myapp"]
          topologyKey: kubernetes.io/hostname

      # Soft rule — PREFER different nodes
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: myapp
            topologyKey: topology.kubernetes.io/zone  # spread across AZs
```

Use `preferred` (soft) for flexibility — with `required` (hard), scheduling fails if no suitable nodes exist (e.g., only 1 node but 3 replicas).

</details>

---

### Q10 🟡 How do namespaces work in Kubernetes and what are they used for?

<details><summary>Click to reveal answer</summary>

Namespaces provide **virtual clusters** within a physical cluster for:
- **Environment isolation**: `dev`, `staging`, `production`
- **Team isolation**: `team-backend`, `team-frontend`
- **Resource quotas**: limit CPU/memory per namespace
- **RBAC scope**: restrict user access to specific namespaces

```bash
# List namespaces
kubectl get namespaces

# Default namespaces:
# default       — default for resources without namespace
# kube-system   — Kubernetes system components
# kube-public   — publicly accessible resources
# kube-node-lease — node heartbeat objects

# Work in a namespace
kubectl get pods -n production
kubectl config set-context --current --namespace=production

# Cross-namespace service DNS:
# <service-name>.<namespace>.svc.cluster.local
jdbc:postgresql://postgres-service.database.svc.cluster.local:5432/myapp
```

Namespaces do NOT isolate network traffic by default — use **NetworkPolicy** for that.

</details>

---

### Q11 🔴 How would you implement a canary deployment in Kubernetes without a service mesh?

<details><summary>Click to reveal answer</summary>

Use two Deployments with the same Service selector but different labels:

```yaml
# Stable deployment (90% traffic via 9 replicas)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp     # Service selects on this
        track: stable
    spec:
      containers:
        - image: myapp:1.0.0

---
# Canary deployment (10% traffic via 1 replica)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp     # Same selector — Service routes to both
        track: canary
    spec:
      containers:
        - image: myapp:2.0.0

---
# Service selects BOTH deployments
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp   # matches both stable and canary pods
  ports:
    - port: 80
      targetPort: 8080
```

Traffic ratio = canary replicas / total replicas = 1/10 = 10%

For precise traffic splitting, use **Istio** or **AWS App Mesh**.

</details>

---

### Q12 🟢 What does `kubectl port-forward` do and when is it useful?

<details><summary>Click to reveal answer</summary>

`kubectl port-forward` tunnels a local port to a port on a pod/service in the cluster. Traffic flows through the kubectl API connection — not the Service.

```bash
# Forward to a pod
kubectl port-forward pod/myapp-abc123 8080:8080 -n production

# Forward to a service
kubectl port-forward svc/myapp-service 8080:80 -n production

# Forward to deployment (picks a pod automatically)
kubectl port-forward deployment/myapp 8080:8080 -n production

# Multiple ports
kubectl port-forward pod/myapp-abc123 8080:8080 5005:5005 -n production
# Access: http://localhost:8080
# Debugger: localhost:5005
```

**Use cases:**
- Access internal services (Prometheus, Grafana, Kibana) without exposing them
- Debug a specific pod's actuator endpoints
- Connect to a database in the cluster
- Remote debugging (JDWP on port 5005)

Not suitable for production traffic — use Services and Ingress.

</details>

---

### Q13 🟡 What are resource requests vs. limits in Kubernetes and why do they matter?

<details><summary>Click to reveal answer</summary>

```yaml
resources:
  requests:
    cpu: "250m"      # guaranteed minimum (scheduler uses this for placement)
    memory: "256Mi"  # guaranteed minimum
  limits:
    cpu: "1000m"     # max allowed (throttled if exceeded)
    memory: "768Mi"  # max allowed (OOMKilled if exceeded)
```

**Requests**: Used by the **scheduler** to find a node with enough available resources. The pod is guaranteed these resources.

**Limits**: Enforced at runtime by the **cgroups** kernel mechanism:
- CPU limit: throttled (slowed down), not killed
- Memory limit: **OOMKilled** (pod killed, restarts)

**Best practices:**
- Always set both requests and limits (required for HPA)
- Set memory limit = 2x request (headroom for GC)
- Set CPU limit = 4x request (burst allowed)
- Monitor actual usage with `kubectl top pods`

**QoS Classes** (priority during resource pressure):
- `Guaranteed`: requests == limits → highest priority
- `Burstable`: limits > requests → medium priority
- `BestEffort`: no requests/limits → evicted first

</details>

---

### Q14 🔴 How do you handle ConfigMap and Secret updates in running pods?

<details><summary>Click to reveal answer</summary>

**Environment variables (`envFrom`)**: NOT updated automatically. Pod must restart.

**Volume mounts**: Updated automatically (kubelet syncs every `configmap-and-secret-change-detection-interval`, default 2 minutes).

```yaml
# Volume mount — auto-updated
spec:
  volumes:
    - name: app-config
      configMap:
        name: myapp-config
  containers:
    - volumeMounts:
        - name: app-config
          mountPath: /config
```

```java
// Spring Cloud Kubernetes — watch for ConfigMap changes
@SpringBootApplication
@EnableConfigurationProperties
public class App {
    // With spring-cloud-kubernetes-client-config dependency,
    // @RefreshScope beans reload when ConfigMap changes
}
```

```yaml
spring:
  cloud:
    kubernetes:
      config:
        enabled: true
        name: myapp-config
        namespace: production
      reload:
        enabled: true
        strategy: refresh   # or restart-context
        period: 30000       # check every 30s
```

For immediate rollout after ConfigMap update:
```bash
kubectl rollout restart deployment/myapp -n production
```

</details>

---

### Q15 🟡 What is a Kubernetes Ingress Controller and how does it differ from a Service?

<details><summary>Click to reveal answer</summary>

| Feature | Service (LoadBalancer) | Ingress + Controller |
|---------|----------------------|---------------------|
| **Cost** | 1 cloud LB per service | 1 cloud LB for all services |
| **Routing** | TCP/UDP, port-based | HTTP/HTTPS, host + path based |
| **TLS termination** | No | Yes |
| **Rate limiting** | No | Yes (Nginx annotations) |
| **Auth** | No | Yes (OAuth2, JWT annotations) |

An **Ingress** is just a Kubernetes resource (rules config). The **Ingress Controller** is a pod that watches Ingress resources and configures the actual load balancer (Nginx, Traefik, AWS ALB Ingress Controller).

```bash
# Install Nginx Ingress Controller (Helm)
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2
```

</details>

---

*End of Chapter 4.3 — Kubernetes Basics for Java Developers*
