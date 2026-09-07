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


























































