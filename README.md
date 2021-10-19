# kube-sticky-session-example

## NGINX

```
To achieve sticky session for both paths you will need two definitions of ingress.

I created example configuration to show you the whole process:

Steps to reproduce:

Apply Ingress definitions
Create deployments
Create services
Create Ingresses
Test
I assume that the cluster is provisioned and is working correctly.

Apply Ingress definitions
Follow this Ingress link to find if there are any needed prerequisites before installing Ingress controller on your infrastructure.

Apply below command to provide all the mandatory prerequisites:

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
Run below command to apply generic configuration to create a service:

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml
Create deployments
Below are 2 example deployments to respond to the Ingress traffic on specific services:

hello.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
      version: 1.0.0
  replicas: 5
  template:
    metadata:
      labels:
        app: hello
        version: 1.0.0
    spec:
      containers:
      - name: hello
        image: "gcr.io/google-samples/hello-app:1.0"
        env:
        - name: "PORT"
          value: "50001"
Apply this first deployment configuration by invoking command:

$ kubectl apply -f hello.yaml

goodbye.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodbye
spec:
  selector:
    matchLabels:
      app: goodbye
      version: 2.0.0
  replicas: 5
  template:
    metadata:
      labels:
        app: goodbye
        version: 2.0.0
    spec:
      containers:
      - name: goodbye 
        image: "gcr.io/google-samples/hello-app:2.0"
        env:
        - name: "PORT"
          value: "50001"
Apply this second deployment configuration by invoking command:

$ kubectl apply -f goodbye.yaml

Check if deployments configured pods correctly:

$ kubectl get deployments

It should show something like that:

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
goodbye   5/5     5            5           2m19s
hello     5/5     5            5           4m57s
Create services
To connect to earlier created pods you will need to create services. Each service will be assigned to one deployment. Below are 2 services to accomplish that:

hello-service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: NodePort
  selector:
    app: hello
    version: 1.0.0
  ports:
  - name: hello-port
    protocol: TCP
    port: 50001
    targetPort: 50001
Apply first service configuration by invoking command:

$ kubectl apply -f hello-service.yaml

goodbye-service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: goodbye-service
spec:
  type: NodePort
  selector:
    app: goodbye
    version: 2.0.0
  ports:
  - name: goodbye-port
    protocol: TCP
    port: 50001
    targetPort: 50001
Apply second service configuration by invoking command:

$ kubectl apply -f goodbye-service.yaml

Take in mind that in both configuration lays type: NodePort

Check if services were created successfully:

$ kubectl get services

Output should look like that:

NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)           AGE
goodbye-service   NodePort    10.0.5.131   <none>        50001:32210/TCP   3s
hello-service     NodePort    10.0.8.13    <none>        50001:32118/TCP   8s
Create Ingresses
To achieve sticky sessions you will need to create 2 ingress definitions.

Definitions are provided below:

hello-ingress.yaml:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "hello-cookie"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/affinity-mode: persistent
    nginx.ingress.kubernetes.io/session-cookie-hash: sha1
spec:
  rules:
  - host: DOMAIN.NAME
    http:
      paths:
      - path: /
        backend:
          serviceName: hello-service
          servicePort: hello-port

goodbye-ingress.yaml:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: goodbye-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "goodbye-cookie"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/affinity-mode: persistent
    nginx.ingress.kubernetes.io/session-cookie-hash: sha1
spec:
  rules:
  - host: DOMAIN.NAME
    http:
      paths:
      - path: /v2/
        backend:
          serviceName: goodbye-service
          servicePort: goodbye-port
Please change DOMAIN.NAME in both ingresses to appropriate to your case. I would advise to look on this Ingress Sticky session link. Both Ingresses are configured to HTTP only traffic.

Apply both of them invoking command:

$ kubectl apply -f hello-ingress.yaml

$ kubectl apply -f goodbye-ingress.yaml

Check if both configurations were applied:

$ kubectl get ingress

Output should be something like this:

NAME              HOSTS        ADDRESS          PORTS   AGE
goodbye-ingress   DOMAIN.NAME   IP_ADDRESS      80      26m
hello-ingress     DOMAIN.NAME   IP_ADDRESS      80      26m
Test
Open your browser and go to http://DOMAIN.NAME Output should be like this:

Hello, world!
Version: 1.0.0
Hostname: hello-549db57dfd-4h8fb
Hostname: hello-549db57dfd-4h8fb is the name of the pod. Refresh it a couple of times.

It should stay the same.

To check if another route is working go to http://DOMAIN.NAME/v2/ Output should be like this:

Hello, world!
Version: 2.0.0
Hostname: goodbye-7b5798f754-pbkbg
Hostname: goodbye-7b5798f754-pbkbg is the name of the pod. Refresh it a couple of times.

It should stay the same.

To ensure that cookies are not changing open developer tools (probably F12) and navigate to place with cookies. You can reload the page to check if they are not changing.
```  

## Appendix

### References

- https://stackoverflow.com/questions/59272484/sticky-sessions-on-kubernetes-cluster
- https://jackiechen.blog/2019/11/06/configure-traefik-sticky-session-in-kubernetes/
