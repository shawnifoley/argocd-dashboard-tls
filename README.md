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
--version v1.12.0 \
--set installCRDs=true
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
