---
title: "Kubernetes SSL Letsencrypt"
date: 2021-06-03T12:53:04-07:00
draft: false
tags: ["kubernetes", "cloud", "docker"]
categories: ["blog"]
authors: ["csgeek"]
---
- [Create An Application](#create-an-application)
- [Load Balancer](#load-balancer)
- [Certificate Manager](#certificate-manager)
  - [Certificate Issuer](#certificate-issuer)
  - [Certificate](#certificate)
- [Update Nginx To Enable SSL](#update-nginx-to-enable-ssl)

Assumptions:
  - You have some familiarity with Docker and Kubernetes
  
**NOTE**: Any yaml file that's included, please apply it via kubectl apply -f file.yml.    
  
Once you have an application up and running you'll need to ensure it's secure.  We'll explore using [letsencrypt](https://letsencrypt.org/) to enable SSL.  It's a free certificate authority and makes it very easy to obtain a certificate.  Naturally SSL doesn't mean your app is secure, but it's a great first step.

LetsEncrypt has two methods to validate the authenticity of the request in order issue a certificate.  

 - HTTP Validation 
 - DNS Validation

Since we're running in K8s, if you do use DNS validation you'll need to use a DNS provider that integrates with K8s.  In order to make this guide more portable and to avoid tying us to any particular DNS provider, I'm going to use HTTP Validation.  

## Create An Application

Before we get started, we just need to create a simple application to test this with.  We'll install a hello world app running 3 replicas and a service load balancer for port 80.  Here's the yaml I've used below.



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world
  name: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  strategy: {}
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - image: docker.io/lsizani/hello
          imagePullPolicy: Always
          name: hello-world
          ports:
            - containerPort: 80
          resources: {}
      restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello-world
  name: hello-docker-svc

  namespace: default

spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: hello-world
  sessionAffinity: None
  type: ClusterIP

status:
  loadBalancer: {}
```

Now, the IP address of the service is still internal so you won't be able to connect to it unless you run the following command temporarily.

```sh
kubectl port-forward svc/hello-docker-svc 8000:80
```

After which you can connect to [http://127.0.0.1:8000](http://127.0.0.1:8000) and you'll see something along these lines:


![ScreenShot](/images/kubernetes-ssl-letsencrypt/screen1.png)

## Load Balancer

At this point we need to actually expose this to the public.  You'll need to install an nginx-ingress Load balancer.

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx ingress-nginx/ingress-nginx --set controller.publishService.enabled=true 
```

Configure nginx to serve traffic on port 80.  Please note the lack of SSL.  We'll revisit this configuration and add SSL in a later pass.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-docker
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"

spec:
  rules:
    - host: hello.demo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-docker-svc
                port:
                  number: 80
```                  

I'm using the DNS name hello.demo.com which is just an example, but if your K8s cluster is configured correctly you'll get provisioned an IP address.  Please update the config to use your own DNS value and have it match the IP address.

For example, if hello.demo.com doesn't resolve to your external IP address this won't work and it'll fail to generate an SSL.

At this point if we navigate to http://hello.demo.com we'll see the nice little hello world screen we've seen previously.


![ScreenShot](/images/kubernetes-ssl-letsencrypt/screen1.png)

## Certificate Manager

Now, we need to add a certificate manager.  This is where you would either choose to use DNS validation.  As I mentioned in the intro, I won't be using DNS validation to make this guide agnostic to the platform.  If you do simply specify the correct options you install the helm chart.

Installing the cert-manager:

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.3.1 \
  --set installCRDs=true
```

### Certificate Issuer

There's a Prod and Staging environment for LetsEncrypt.  I installed both of them for convenience.  Staging isn't rate limited and allows you to debug/validate behavior without risking getting hit by a quota.  

**Staging**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@email.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-staging-ssl
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
      - http01:
          ingress:
            class: nginx


```

**Production**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: samir@es.net
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-prod
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
      - http01:
          ingress:
            class: nginx
```

At this point we have the ability to create a certificate.  It will utilize which ever issuer we want.  In the example below, I'm using the letsencrypt-prod which matches the definition above.

### Certificate 

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  namespace: default
  name: hello-certs
spec:
  secretName: hello-certs
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - hello.demo.com
```

If everything worked correctly you should be able to see the certificate via

```sh
kubectl get secrets hello-certs -o json
```

## Update Nginx To Enable SSL

Finally now that we have a certificate, let's enable nginx to use it.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-docker
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:
    - hosts:
        - hello.demo.com
      secretName: hello-certs
  rules:
    - host: hello.demo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-docker-svc
                port:
                  number: 80
```

At this point you should be able to connect https://hello.demo.com and get a valid response.

You can also check the validity of your certificate by testing it via [SSL Labs ](https://www.ssllabs.com/ssltest/) I received in A+ for the certificate I generated