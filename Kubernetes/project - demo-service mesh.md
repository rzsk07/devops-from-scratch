Real-World Project

                          USER
                           │
                           ▼
                  Istio Ingress Gateway
                           │
                           ▼
                     frontend Service
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              frontend Pod    Envoy Proxy
                    │
                    │
                    ▼
              reviews Service
                    │
             ┌──────┴──────┐
             ▼             ▼
        reviews-v1      reviews-v2
        App + Envoy     App + Envoy
             │             │
             └──────┬──────┘
                    │
                    ▼
             backend Service
                    │
                    ▼
             backend Pod
                + Envoy
Istio architecture:

              CONTROL PLANE
              ┌─────────────┐
              │   istiod    │
              └──────┬──────┘
                     │
          xDS configuration
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
   Envoy           Envoy          Envoy
 frontend        reviews-v1     reviews-v2
      │              │              │
      └──────────────┴──────────────┘
                  DATA PLANE


===========================================================================================

kubectl get nodes

# Istio install:-

Current Istio release documentation is Istio 1.31, and Istio's official getting-started guide uses the demo profile for evaluation/testing.

Download Istio

EC2 Ubuntu:

curl -L https://istio.io/downloadIstio | sh -


LAB 1 — Install Istio
1.1 Install istioctl

EC2:

curl -sL https://istio.io/downloadIstioctl | sh -

PATH:

export PATH=$HOME/.istioctl/bin:$PATH

Permanent:

echo 'export PATH=$HOME/.istioctl/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

Verify:

which istioctl
istioctl version
1.2 Install Istio

For our learning environment:

istioctl install --set profile=demo -y

Check:

kubectl get pods -n istio-system

You should see istiod running.

Then:

istioctl verify-install

And:

istioctl version
LAB 1 success
istiod       Running
istioctl     working
verify       successful

LAB 1 complete hone ke baad hi LAB 2.

LAB 2 — Create mesh-lab

Create:

mkdir -p ~/istio-lab
cd ~/istio-lab

File:

namespace.yaml

Content:

apiVersion: v1
kind: Namespace
metadata:
  name: mesh-lab

Apply:

kubectl apply -f namespace.yaml

Verify:

kubectl get namespace mesh-lab
LAB 3 — Enable Sidecar Injection

Create:

sidecar-injection.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mesh-lab
  labels:
    istio-injection: enabled

Apply:

kubectl apply -f sidecar-injection.yaml

Verify:

kubectl get namespace mesh-lab --show-labels

You should see:

istio-injection=enabled
Important concept

Label lagane se existing Pods mein automatically Envoy nahi aata.

Injection Pod creation time par hoti hai.

Isliye:

kubectl delete pods -n mesh-lab --all

Abhi agar Pods hain tabhi delete karna; fresh deployment kar rahe ho to zaroori nahi.

LAB 4 — Deploy Application

Hum intentionally application ko simple rakhenge:

frontend
   ↓
reviews
   ├── v1
   └── v2

frontend
   ↓
backend

Lekin ek important improvement:

frontend application aur traffic generator ko separate rakhenge.

4.1 backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: mesh-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Hello from BACKEND v1"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: mesh-lab
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 5678

Apply:

kubectl apply -f backend.yaml
LAB 4.2 — Reviews

File:

reviews.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v1
  namespace: mesh-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reviews
      version: v1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      containers:
        - name: reviews
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Reviews Service - VERSION 1"
          ports:
            - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v2
  namespace: mesh-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reviews
      version: v2
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      containers:
        - name: reviews
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Reviews Service - VERSION 2"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  selector:
    app: reviews
  ports:
    - name: http
      port: 80
      targetPort: 5678

Apply:

kubectl apply -f reviews.yaml
LAB 4.3 — Frontend

File:

frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: mesh-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Hello from FRONTEND"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: mesh-lab
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 5678

Apply:

kubectl apply -f frontend.yaml
LAB 4.4 — Traffic Generator

This Pod continuously calls our services.

File:

traffic-generator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-generator
  namespace: mesh-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traffic-generator
  template:
    metadata:
      labels:
        app: traffic-generator
    spec:
      containers:
        - name: curl
          image: curlimages/curl:8.10.1
          command:
            - /bin/sh
            - -c
            - |
              while true
              do
                echo "===== BACKEND ====="
                curl -s http://backend.mesh-lab.svc.cluster.local
                echo

                echo "===== REVIEWS ====="
                curl -s http://reviews.mesh-lab.svc.cluster.local
                echo

                sleep 5
              done

Apply:

kubectl apply -f traffic-generator.yaml
LAB 5 — Verify 2/2 Pods

Now:

kubectl get pods -n mesh-lab

Expected:

NAME                                  READY
backend-xxxxx                        2/2
backend-yyyyy                        2/2
reviews-v1-xxxxx                     2/2
reviews-v1-yyyyy                     2/2
reviews-v2-xxxxx                     2/2
reviews-v2-yyyyy                     2/2
frontend-xxxxx                       2/2
traffic-generator-xxxxx              2/2

Why 2/2?

Pod
├── Application container
└── istio-proxy (Envoy)

Verify:

kubectl get pod -n mesh-lab -o wide

And:

kubectl describe pod -n mesh-lab \
  $(kubectl get pod -n mesh-lab -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')
LAB 6 — Envoy Logs

Get frontend Pod:

FRONTEND_POD=$(kubectl get pod -n mesh-lab \
  -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')

echo $FRONTEND_POD

Application logs:

kubectl logs -n mesh-lab $FRONTEND_POD -c frontend

Envoy logs:

kubectl logs -n mesh-lab $FRONTEND_POD -c istio-proxy

This is where you understand:

Application
     │
     ▼
Envoy
     │
     ▼
Network
LAB 7 — istioctl proxy-status

Now that istioctl is installed:

istioctl proxy-status

You want to see proxies associated with your Pods.

Also:

istioctl analyze -n mesh-lab

If there are warnings, don't ignore them. We'll understand what each warning means.

LAB 8 — DestinationRule

Now we introduce:

DestinationRule

Purpose:

Define policies/subsets for a destination.

Create:

destination-rules.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  host: reviews.mesh-lab.svc.cluster.local

  subsets:
    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2

Apply:

kubectl apply -f destination-rules.yaml

Verify:

kubectl get destinationrule -n mesh-lab

And:

kubectl describe destinationrule reviews -n mesh-lab
LAB 9 — VirtualService

Now we tell Istio:

When someone requests reviews, where should traffic go?

Create:

virtual-service.yaml

Initial configuration:

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  hosts:
    - reviews.mesh-lab.svc.cluster.local

  http:
    - route:
        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v1

Apply:

kubectl apply -f virtual-service.yaml

Test:

kubectl exec -n mesh-lab \
  deploy/traffic-generator \
  -c curl -- \
  curl -s http://reviews.mesh-lab.svc.cluster.local

You should consistently get:

Reviews Service - VERSION 1
LAB 10 — 80/20 Traffic Split

Now replace virtual-service.yaml with:

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  hosts:
    - reviews.mesh-lab.svc.cluster.local

  http:
    - route:

        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v1
          weight: 80

        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v2
          weight: 20

Apply:

kubectl apply -f virtual-service.yaml

Generate traffic:

for i in $(seq 1 50); do
  kubectl exec -n mesh-lab deploy/traffic-generator -c curl -- \
    curl -s http://reviews.mesh-lab.svc.cluster.local
done

Concept:

                 reviews
                    │
          ┌─────────┴─────────┐
          │                   │
        80%                   20%
          │                   │
          ▼                   ▼
      reviews-v1          reviews-v2

It won't necessarily produce exactly 40/10 results in 50 requests. Weight is traffic distribution, not a strict counter.

LAB 11 — mTLS STRICT

Create:

peer-authentication.yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mesh-lab
spec:
  mtls:
    mode: STRICT

Apply:

kubectl apply -f peer-authentication.yaml

Verify:

kubectl get peerauthentication -n mesh-lab

Concept:

Frontend Envoy
      │
      │ mTLS
      │
      ▼
Reviews Envoy
      │
      ▼
Reviews application

The application itself doesn't need to implement mTLS.

LAB 12 — AuthorizationPolicy

For clean security practice, let's create dedicated ServiceAccounts.

File:

service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend
  namespace: mesh-lab
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reviews
  namespace: mesh-lab
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: mesh-lab

Apply:

kubectl apply -f service-accounts.yaml

Then update frontend.yaml:

spec:
  template:
    metadata:
      labels:
        app: frontend
    spec:
      serviceAccountName: frontend

And backend:

spec:
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      serviceAccountName: backend

Reviews:

spec:
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      serviceAccountName: reviews

Do the same for reviews-v2.

Then recreate Pods:

kubectl apply -f frontend.yaml
kubectl apply -f backend.yaml
kubectl apply -f reviews.yaml

Now AuthorizationPolicy:

authorization-policy.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: mesh-lab
spec:
  selector:
    matchLabels:
      app: reviews

  action: ALLOW

  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/mesh-lab/sa/frontend

Apply:

kubectl apply -f authorization-policy.yaml

Meaning:

frontend
   │
   │ ALLOWED
   ▼
reviews

other identity
   │
   │ DENIED
   ▼
reviews

This is much better than allowing the namespace's generic default ServiceAccount.

LAB 13 — Timeout

Modify virtual-service.yaml.

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  hosts:
    - reviews.mesh-lab.svc.cluster.local

  http:
    - timeout: 2s

      route:
        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v1

Apply:

kubectl apply -f virtual-service.yaml

Concept:

Request
   │
   ▼
Envoy
   │
   ├── 0s
   ├── 1s
   └── 2s → TIMEOUT

Timeout is controlled by Envoy rather than requiring the application to implement it.

LAB 14 — Retry

Now replace the http: section with:

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  hosts:
    - reviews.mesh-lab.svc.cluster.local

  http:
    - retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,connect-failure,reset

      route:
        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v1

Apply:

kubectl apply -f virtual-service.yaml

Concept:

              Envoy
                │
                ▼
          Attempt #1
              ❌
                │
              retry
                ▼
          Attempt #2
              ❌
                │
              retry
                ▼
          Attempt #3
              ✅

Again, retries happen only when the configured retry conditions occur.

LAB 15 — Circuit Breaking

Important: Don't create a second conflicting DestinationRule for the same host.

Replace destination-rules.yaml temporarily with:

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  host: reviews.mesh-lab.svc.cluster.local

  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1

      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1

    outlierDetection:
      consecutive5xxErrors: 2
      interval: 5s
      baseEjectionTime: 30s
      maxEjectionPercent: 100

  subsets:
    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2

Apply:

kubectl apply -f destination-rules.yaml

Concept:

Many requests
      │
      ▼
   Envoy
      │
      ├── healthy → backend
      │
      └── unhealthy
              │
              ▼
        circuit protection

We'll later deliberately generate failures to see outlier detection/ejection.

LAB 16 — Fault Injection

For this lab, temporarily replace virtual-service.yaml with:

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: mesh-lab
spec:
  hosts:
    - reviews.mesh-lab.svc.cluster.local

  http:
    - fault:
        delay:
          percentage:
            value: 100
          fixedDelay: 5s

      route:
        - destination:
            host: reviews.mesh-lab.svc.cluster.local
            subset: v1

Apply:

kubectl apply -f virtual-service.yaml

Test:

time kubectl exec -n mesh-lab deploy/traffic-generator \
  -c curl -- \
  curl -s http://reviews.mesh-lab.svc.cluster.local

You should observe approximately a 5-second delay.

Concept:

Client
  │
  ▼
Envoy
  │
  │ inject artificial delay
  ▼
Reviews

After the lab, restore your normal virtual-service.yaml.

Don't combine this blindly with your retry/timeout configuration. Fault injection has specific interactions with retry/timeout behavior, so we'll test those as separate controlled configurations.

LAB 17 — Prometheus

First check:

kubectl get pods -n istio-system

If Prometheus isn't present, from your Istio installation directory:

kubectl apply -f samples/addons/prometheus.yaml

Then:

kubectl get pods -n istio-system

Open:

istioctl dashboard prometheus

You can inspect Istio metrics such as:

istio_requests_total

Traffic:

Frontend
   │
 Envoy
   │
   ▼
Reviews
   │
 Envoy
   │
   ▼
Prometheus
LAB 18 — Grafana

Install if necessary:

kubectl apply -f samples/addons/grafana.yaml

Check:

kubectl get pods -n istio-system

Open:

istioctl dashboard grafana

Grafana consumes Prometheus metrics:

Envoy
  │
  ▼
Prometheus
  │
  ▼
Grafana
LAB 19 — Kiali

Install:

kubectl apply -f samples/addons/kiali.yaml

Check:

kubectl get pods -n istio-system

Open:

istioctl dashboard kiali

Kiali gives us a visual service graph:

             frontend
                 │
          ┌──────┴──────┐
          ▼             ▼
       backend        reviews
                         │
                    ┌────┴────┐
                    ▼         ▼
                   v1        v2

This is particularly useful for understanding service-to-service communication.

LAB 20 — Envoy proxy-config

This is one of the most important labs.

Find frontend:

FRONTEND_POD=$(kubectl get pod -n mesh-lab \
  -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')

echo $FRONTEND_POD
Listeners
istioctl proxy-config listeners \
  $FRONTEND_POD \
  -n mesh-lab
Routes
istioctl proxy-config routes \
  $FRONTEND_POD \
  -n mesh-lab
Clusters
istioctl proxy-config clusters \
  $FRONTEND_POD \
  -n mesh-lab
Endpoints
istioctl proxy-config endpoints \
  $FRONTEND_POD \
  -n mesh-lab
TLS certificates/secrets
istioctl proxy-config secret \
  $FRONTEND_POD \
  -n mesh-lab

This lets you see what's actually inside Envoy.

📁 Final Project Directory

Aapka complete project eventually:

~/istio-lab/
│
├── namespace.yaml
│
├── sidecar-injection.yaml
│
├── service-accounts.yaml
│
├── backend.yaml
│
├── reviews.yaml
│
├── frontend.yaml
│
├── traffic-generator.yaml
│
├── destination-rules.yaml
│
├── virtual-service.yaml
│
├── peer-authentication.yaml
│
└── authorization-policy.yaml























































