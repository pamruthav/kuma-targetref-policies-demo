```
kubectl delete meshcircuitbreaker mesh-circuit-breaker-all-default -n kuma-system
kubectl delete meshtimeout mesh-gateways-timeout-all-default -n kuma-system
kubectl delete meshtimeout mesh-timeout-all-default -n kuma-system
kubectl delete meshretry mesh-retry-all-default -n kuma-system
```

### MeshCircuitBreaker
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshCircuitBreaker
metadata:
  name: frontend-to-backend-circuit-breaker
  namespace: kuma-system
  labels:
    kuma.io/mesh: default
spec:
  targetRef:
    kind: MeshService
    name: frontend_kuma-demo_svc_8080
  to:
  - targetRef:
      kind: MeshService
      name: backend_kuma-demo_svc_3001
    default:
      connectionLimits:
        maxConnections: 2
        maxPendingRequests: 8
        maxRetries: 2
        maxRequests: 2
EOF
```

### MeshFaultInjection
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshFaultInjection
metadata:
  name: frontend-to-backend-fault-injection
  namespace: kuma-system
spec:
  targetRef:
    kind: MeshService
    name: backend_kuma-demo_svc_3001
  from:
    - targetRef:
        kind: MeshService
        name: frontend_kuma-demo_svc_8080
      default:
        http:
          - abort:
              httpStatus: 500
              percentage: 50
EOF
```


### MeshRateLimit
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshRateLimit
metadata:
  name: backend-rate-limit
  namespace: kuma-system
spec:
  targetRef:
    kind: MeshService
    name: backend_kuma-demo_svc_3001
  from:
    - targetRef:
        kind: Mesh
      default:
        local:
          http:
            requestRate:
              num: 5
              interval: 10s
            onRateLimit:
              status: 429
              headers:
                set:
                  - name: "x-kuma-rate-limited"
                    value: "true"
EOF
```
### enable mTLS before next policy
```
cat <<EOF | kubectl apply -f - 
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin

EOF
```
### MeshTrafficPermission

#### deny-all
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshTrafficPermission
metadata:
  namespace: kuma-system
  name: deny-all
  labels:
    kuma.io/mesh: default
spec:
  targetRef: 
    kind: Mesh
  from:
    - targetRef: 
        kind: Mesh
      default: 
        action: Deny
EOF
```
#### allow fe-be
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshTrafficPermission
metadata:
  namespace: kuma-system
  name: allow-frontend-to-backend
spec:
  targetRef:
    kind: MeshService
    name: backend_kuma-demo_svc_3001
  from:
    - targetRef:
        kind: MeshService
        name: frontend_kuma-demo_svc_8080
      default:
        action: Allow
EOF
```
``` kubectl -n kuma-system delete meshtrafficpermissions deny-all```

#### allow-all
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: MeshTrafficPermission
metadata:
  namespace: kuma-system
  name: allow-all
  labels:
    kuma.io/mesh: default
spec:
  targetRef: 
    kind: Mesh
  from:
    - targetRef: 
        kind: Mesh
      default: 
        action: Allow
EOF
```

---------------------

## Demo-2

```shell
export PROXY_IP=$(kubectl get svc --namespace microservice-mesh edge-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $PROXY_IP


```shell
curl -s http://$PROXY_IP/ | jq .
```


## Play with MeshTimeout policy

### Add latency to svc-000:

```shell
kubectl edit configmap -n microservice-mesh api-play-000
```

add: `"latency": {"min_millis": 1000, "max_millis": 2000},` 

see it's longer: `curl -s http://$PROXY_IP/`

### Apply a timeout that applies to all DPPs

```shell
kubectl apply -f timeout-1.yaml
```

- See it fail: `curl -s http://$PROXY_IP/`
- Apply the timeout to fix it `kubectl apply -f timeout-2.yaml`
- See it work: `curl -s http://$PROXY_IP/`
- Look at the GUI

### Now more latency to svc-003!

```shell
kubectl edit configmap -n microservice-mesh api-play-003
```

add: `"latency": {"min_millis": 1000, "max_millis": 2000},`

see it fails: `curl -s http://$PROXY_IP/`

### Try to fix it

- Apply the timeout to fix it `kubectl apply -f timeout-3.yaml`
- See it still fails: `curl -s http://$PROXY_IP/`

It's because svc-000 doesn't talk directly to svc-003 we need to add 2 timeouts one from svc-000 to svc-002 and one from svc-002 to svc-003

- Apply the timeout to fix it `kubectl apply -f timeout-3-fixed.yaml`
- See it work: `curl -s http://$PROXY_IP/`
- Look at the GUI
