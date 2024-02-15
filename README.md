# Introduction 

This guide outlines the process for configuring and enabling Let's Encrypt and Certificate Manager within a Kubernetes cluster. By following these steps, you'll facilitate secure communication between ingress controllers and the internet, automating the management and issuance of SSL/TLS certificates.


# Prerequisite

- A Kubernetes cluster with Traefik and Nginx ingress controllers installed.

# Installation Steps

## 1. Install Certificate Manager

Install cert-manager to issue Let's Encrypt certificates by applying the latest version:

```kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.1/cert-manager.yaml```

Verify the installation by ensuring that all cert-manager pods are running:

```kubectl get pods -n cert-manager --watch```



## 2. Configure the ClusterIssuer
The ClusterIssuer resource automates SSL/TLS certificate issuance. Configure it for both Nginx and Traefik as follows:

### Nginx ClusterIssuer
```
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: cert-manager
spec:
  acme:
    email: letsencrypt.func@test.com
    preferredChain: ''
    privateKeySecretRef:
      name: letsencrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: nginx           

```

### Traefik ClusterIssuer
Similar to Nginx, but with ingress.class: traefik.
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-traefik
  namespace: cert-manager
spec:
  acme:
    email: letsencrypt.func@test.com
    preferredChain: ''
    privateKeySecretRef:
      name: letsencrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik

```            


## 3. Usage

Deploy sample applications and configure ingress to use the issued certificates by creating Kubernetes Deployment, Service, and Ingress (or IngressRoute for Traefik) resources.

### Deployment Example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    appType: web 
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 1
  template:
    metadata:
      labels:
        app: myapp
        appType: web
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          securityContext:
            privileged: true             
      imagePullSecrets:
        - name: myregistry-secret
      restartPolicy: Always
      terminationGracePeriodSeconds: 0
```

```
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: myapp

```

### Ingress Configuration:

Create Ingress resources for Nginx and Traefik.

#### Nginx
```
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "myapp"
    nginx.ingress.kubernetes.io/session-cookie-expires: "7200"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "7200"  
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
    cert-manager.io/cluster-issuer: "letsencrypt"
    kubernetes.io/ingress.class: nginx  
spec:
  tls:
    - hosts:
        - myapp-nginx.com
      secretName: myapp-nginx-tls
  rules:
    - host: myapp-nginx.com
      http:
        paths:
          - backend:
              service:
                name: myapp-service
                port:
                  number: 80
            path: /
            pathType: Prefix

```

#### Traefik

```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`myapp-traefik.com`)
      services:
        - name: myapp
          port: 80
  tls:
    secretName: myapp-traefik-tls

```
### Creating Certificates
After configuring your ClusterIssuer, create a Certificate resource to specify how you want the certificates to be issued. Here's an example for a domain using the previously configured 

#### Nginx

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-nginx
spec:
  # Use this tls secret in the ingress
  secretName: myapp-nginx-tls
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  dnsNames:
    # Use this DNS name in ingress
    - "myapp-nginx.com"

```
### Trafik

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-traefik
spec:
  # Use this tls secret in the ingress
  secretName: myapp-traefik-tls
  issuerRef:
    name: letsencrypt-traefik
    kind: ClusterIssuer
  dnsNames:
    # Use this DNS name in ingress
    - "myapp-traefik.com"

```