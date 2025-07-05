**Install Nginx Ingress Controller**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
--namespace ingress-nginx \
--create-namespace \
--set controller.publishService.enabled=true
```

Check the installation:

```bash
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx  # Look for the LoadBalancer service
```

**Install cert-manager**

cert-manager automates the process of obtaining and renewing TLS certificates from various sources, including Let's Encrypt.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.17.1 \
--set crds.enabled=true
```

Verify installation:

```bash
kubectl get pods -n cert-manager
```

**Create a Kubernetes Secret**

```bash
kubectl create secret generic cloudflare-api-token-secret \
--namespace=cert-manager \
--from-literal=api-token="YOUR_CLOUDFLARE_API_TOKEN"
```

**Create a ClusterIssuer or Issuer**

Now, create a `ClusterIssuer` (or `Issuer` if you want namespace-scoped configuration) that uses the Cloudflare DNS-01 solver.

```yaml
#cluster-issuer-cloudflare.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: shawnifoley@gmail.com
    privateKeySecretRef:
      name: letsencrypt-cloudflare-private-key
    solvers:
      - dns01:
          cloudflare:
            email: shawnifoley@gmail.com
            apiTokenSecretRef:
              name: cloudflare-api-token-secret # Name of the Secret we created
              key: api-token # The key within the Secret
        selector:
          dnsZones:
            - "fol3y.us"
```

Apply it:

```bash
kubectl apply -f cluster-issuer-cloudflare.yaml
```

**Use the ClusterIssuer in your Ingress**
Update your Argo CD Ingress (or any other Ingress) to use this `ClusterIssuer`.

```yaml
#argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - argocd.fol3y.us
      secretName: letsencrypt-cloudflare-private-key
  rules:
    - host: argocd.fol3y.us
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

Apply the changes:

```bash
kubectl apply -f argocd-ingress.yaml
```

**Check Certificate Status:**

```bash
# Check the certificate status
kubectl get certificate -n argocd
kubectl describe certificate -n argocd

# Check the challenge status
kubectl get challenge -n argocd
kubectl describe challenge -n argocd

# Check the order status
kubectl get order -n argocd
kubectl describe order -n argocd
```

**Get default admin password**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
# or with argocd cli
argocd admin initial-password -n argocd
```

**Reset default admin password**
```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
kubectl create secret generic argocd-initial-admin-secret --from-literal=password=<new_password> -n argocd
# or with argocd cli
argocd account update-password
```

**Deploy demo-apps**
```bash
argocd app create argo-demo --repo https://github.com/shawnifoley/argocd-demo.git   --path . --dest-server https://kubernetes.default.svc --dest-namespace argo-demo
```

**Checking ArgoCD Installation in Kubernetes**
## 1. Check if ArgoCD Namespace Exists

```bash
kubectl get namespace argocd
```

## 2. Verify ArgoCD Components

Check if the core ArgoCD pods are running:
```bash
kubectl get pods -n argocd
```

You should see several pods including:
- argocd-application-controller
- argocd-dex-server
- argocd-redis
- argocd-repo-server
- argocd-server

## 3. Check ArgoCD Deployments

```bash
kubectl get deployments -n argocd
```

## 4. Check ArgoCD Services

```bash
kubectl get services -n argocd
```

The service `argocd-server` is the main API and UI server.

## 5. Get ArgoCD Server URL

If using LoadBalancer or NodePort:
```bash
kubectl get svc argocd-server -n argocd
```

## 6. Get Initial Admin Password

For ArgoCD versions before 2.x:
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

For ArgoCD versions 2.x+:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 7. Check ArgoCD Version

```bash
kubectl exec -it -n argocd deployment/argocd-server -- argocd version
```

## 8. Verify ArgoCD CLI (if installed locally)

If you have the ArgoCD CLI installed:
```bash
argocd version
```

## 9. Check ArgoCD Applications

```bash
kubectl get applications -n argocd
```

Or using ArgoCD CLI:
```bash
argocd app list
```

## 10. Check ArgoCD Configuration

```bash
kubectl get configmap argocd-cm -n argocd
kubectl get configmap argocd-rbac-cm -n argocd
```

## 11. Check ArgoCD TLS Configuration

```bash
kubectl get secret letsencrypt-cloudflare-private-key -n argocd
```

# Verifying cert-manager Installation in Kubernetes

## 1. Check if cert-manager Namespace Exists

```bash
kubectl get namespace cert-manager
```

## 2. Verify cert-manager Pods

Check if all cert-manager pods are running:
```bash
kubectl get pods -n cert-manager
```

You should see several pods running:
- cert-manager (the main controller)
- cert-manager-cainjector
- cert-manager-webhook

All pods should be in the "Running" state.

## 3. Check cert-manager Deployments

```bash
kubectl get deployments -n cert-manager
```

Verify all deployments have the desired number of replicas available.

## 4. Check cert-manager CRDs (Custom Resource Definitions)

cert-manager installs several CRDs that are essential for its operation:
```bash
kubectl get crd | grep cert-manager
```

You should see CRDs including:
- certificates.cert-manager.io
- certificaterequests.cert-manager.io
- challenges.acme.cert-manager.io
- clusterissuers.cert-manager.io
- issuers.cert-manager.io
- orders.acme.cert-manager.io

## 5. Check cert-manager Version

```bash
kubectl -n cert-manager get deployment cert-manager -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## 6. Verify Issuers and ClusterIssuers

Check for any configured Issuers:
```bash
kubectl get issuers --all-namespaces
```

Check for any configured ClusterIssuers:
```bash
kubectl get clusterissuers
```

## 7. Check for Existing Certificates

```bash
kubectl get certificates --all-namespaces
```

## 8. Test cert-manager with a Self-Signed Certificate

You can test cert-manager by creating a test issuer and certificate:

1. Create a test issuer YAML file (test-issuer.yaml):
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: default
spec:
  selfSigned: {}
```

2. Apply the issuer:
```bash
kubectl apply -f test-issuer.yaml
```

3. Create a test certificate YAML file (test-certificate.yaml):
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: default
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
    kind: Issuer
```

4. Apply the certificate:
```bash
kubectl apply -f test-certificate.yaml
```

5. Check if the certificate was issued:
```bash
kubectl get certificate -n default selfsigned-cert
```

6. Verify the secret was created:
```bash
kubectl get secret -n default selfsigned-cert-tls
```
