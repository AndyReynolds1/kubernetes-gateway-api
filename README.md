# Kubernetes Gateway API

A quick look at the new Kubernetes Gateway API that was recently announced.

https://gateway-api.sigs.k8s.io/
https://gateway-api.sigs.k8s.io/guides/simple-gateway/
https://github.com/nginxinc/nginx-gateway-fabric/blob/main/examples/cafe-example/gateway.yaml

## Basic Example

Create namespace:

```bash
kubectl create namespace web
```

Create simple deployment with sample website:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: web
  name: demo-website
  labels:
    app: demo-website
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-website
  template:
    metadata:
      labels:
        app: demo-website
    spec:
      containers:
      - name: web
        image: areynolds762/demo-node-website:1.0
        ports:
        -  containerPort: 8080
EOF
```

Add internal only service of type `ClusterIP`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  namespace: web
  name: demo-website
  labels:
    app: demo-website
spec:
  type: ClusterIP
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: demo-website
EOF
```

Website should now be running but not accessible from outside the cluster.

## Gateway API

Example using NGINX Gateway Fabric: https://github.com/nginxinc/nginx-gateway-fabric/blob/main/docs/installation.md

Install the Gateway API resources from the standard channel (the CRDs and the validating webhook):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.8.1/standard-install.yaml
```

Deploy the NGINX Gateway Fabric CRDs:

```bash
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.0.0/crds.yaml
```

Deploy the NGINX Gateway Fabric:

```bash
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.0.0/nginx-gateway.yaml
```

Confirm the NGINX Gateway Fabric is running in nginx-gateway namespace:

```bash
kubectl get pods -n nginx-gateway
```

Expose NGINX Gateway Fabric - NodePort example
```bash
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.0.0/deploy/manifests/service/nodeport.yaml
```

Check port number used by NodePort service.  Should be called `nginx-gateway` in the `nginx-gateway` namespace.

Create Gateway:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  namespace: web
  name: gateway-web
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
	allowedRoutes:
      namespaces:
        from: Same
EOF
```

Create HTTPRoute:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  namespace: web
  name: httproute-web
spec:
  parentRefs:
  - name: gateway-web
  rules:
  - backendRefs:
    - name: demo-website
      port: 8080
EOF
```

Sample website should now be accessible outside the cluster using HTTP port used in the `nginx-gateway` service.
