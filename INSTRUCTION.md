# Kubernetes Ingress Validation Instructions

This document provides step-by-step instructions to validate the Kubernetes Ingress configuration for the Django ToDo list application.

## Prerequisites

- `kind` cluster management tool installed
- `kubectl` command-line tool installed
- Docker running
- The application repository cloned and ready

## Validation Steps

### 1. Create the Kind Cluster

```bash
kind create cluster --config cluster.yml
```

Expected output should show:
- Cluster creation successful
- kubectl context set to "kind-kind"

### 2. Deploy the Application and Ingress

Run the bootstrap script to deploy the MySQL database, the Django ToDo app, and the Ingress controller:

```bash
bash bootstrap.sh
```

Expected output should show all resources created/configured:
- MySQL namespace, configmap, secret, service, and statefulset
- ToDo app namespace, volumes, secrets, configmap, services, HPA, and deployment
- Ingress-NGINX controller installation
- Ingress resource creation: `ingress.networking.k8s.io/todoapp-ingress created`

### 3. Wait for Pods to Be Ready

Monitor the deployment until all pods are running:

```bash
# Check todoapp pods
kubectl get pods -n todoapp

# Check MySQL pods
kubectl get pods -n mysql

# Check ingress controller pods
kubectl get pods -n ingress-nginx
```

Expected output:
- All pods in todoapp namespace should show `READY` as `1/1` and `STATUS` as `Running`
- All MySQL pods should be ready
- Ingress nginx controller pod should be running

### 4. Verify Ingress Configuration

Check the Ingress resource:

```bash
kubectl get ingress -n todoapp
kubectl describe ingress todoapp-ingress -n todoapp
```

Expected output should show:
- Ingress name: `todoapp-ingress`
- Class: `nginx`
- Rules with HTTP path `/(.*)` routing to `todoapp-service`
- Address should be assigned (usually `localhost` for kind)

### 5. Access the Application

#### From Inside the Cluster

Run a temporary pod to test connectivity:

```bash
kubectl run test-curl --image=curlimages/curl:latest -it --rm --restart=Never -- curl http://todoapp-service.todoapp.svc.cluster.local/
```

Expected response: HTML page of the Django ToDo app homepage

#### Via Ingress From Inside the Cluster

```bash
kubectl run test-ingress --image=curlimages/curl:latest -it --rm --restart=Never -- curl http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/
```

Expected response: HTML page of the Django ToDo app homepage (should NOT show 404)

#### From Host Machine (Using Port-Forward)

For development/testing, use kubectl port-forward to access the app:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &
curl http://localhost:8080/
```

Expected response:
- HTTP 200 OK
- HTML page of the Django ToDo app homepage

### 6. Test Path-Based Routing

Test various paths to ensure all requests are properly routed:

```bash
# Test root path
curl -s http://localhost:8080/ | grep -c "Djodolist"

# Test API endpoint
curl -s http://localhost:8080/api/ | grep -c "<!DOCTYPE"

# Test static files don't return 404
curl -s http://localhost:8080/static/css/custom.css | grep -c "404" || echo "No 404 error"

# Test login page
curl -s http://localhost:8080/auth/login/ | grep -c "<!DOCTYPE"
```

Expected results:
- All paths should return HTTP 200 (no 404 errors)
- All paths should return HTML content (not 404 error page)

### 7. Check Ingress Annotations and Configuration

Verify the ingress is properly configured with regex support:

```bash
kubectl get ingress todoapp-ingress -n todoapp -o yaml
```

Expected configuration should include:
- `nginx.ingress.kubernetes.io/use-regex: "true"` - enables regex path matching
- `nginx.ingress.kubernetes.io/rewrite-target: /$1` - rewrites paths while preserving captured path segment
- Path specification with regex: `/(.*)` - captures all paths
- Backend service: `todoapp-service` on port 80

### 8. Verify No 404 Errors in Browser Console

When accessing the app in a browser via the ingress:

```bash
# Start port-forward
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &

# Open in default browser
$BROWSER http://localhost:8080/

# Stop port-forward when done
kill %1
```

Expected results:
- Page loads without 404 errors
- All static assets (CSS, JS) load successfully
- No failed requests in browser developer tools console
- Login link and other navigation elements work

## Troubleshooting

### Pods not starting
- Check pod logs: `kubectl logs -n todoapp <pod-name>`
- Ensure MySQL pods are ready first
- Wait for images to be pulled

### Ingress not routing traffic
- Verify ingress is created: `kubectl get ingress -n todoapp`
- Check ingress logs: `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx`
- Verify service exists: `kubectl get svc -n todoapp`

### Cannot access via HTTP on port 80
- In dev containers or behind NAT, use port-forward instead
- Verify port mappings in kind cluster configuration
- Check if ingress controller pod is running: `kubectl get pods -n ingress-nginx`

## Cleanup

To remove the kind cluster:

```bash
kind delete cluster
```

This removes all resources and the local Kubernetes cluster.
