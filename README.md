# kube-sticky-session-example

### Steps to reproduce:

- Create deployments
- Apply prerequisites (nginx|traefik)
- Create services (nginx|traefik)
- Create Ingresses (nginx|traefik)
- Test

I assume that the cluster is provisioned and is working correctly.

## Common App Deployment

We will schedule our two apps with the following command and manifest.

`kubectl apply -f https://raw.githubusercontent.com/kempy007/kube-sticky-session-example/main/common/deployments.yaml`

Check if deployments configured pods correctly:

`$ kubectl -n simple-apps get deployments`

It should show something like that:

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
goodbye   5/5     5            5           2m19s
hello     5/5     5            5           4m57s
```

--------
## NGINX

### Apply prerequisites

Apply below command to provide all the mandatory prerequisites for nginx:

`$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml`


### Create services

To connect to earlier created pods you will need to create services. Each service will be assigned to one deployment. Below are 2 services to accomplish that:

Apply these services by invoking command:

`$ kubectl apply -f https://raw.githubusercontent.com/kempy007/kube-sticky-session-example/main/nginx/services.yaml`

##### Take in mind that in both configuration specifies type: `NodePort`

Check if services were created successfully:

`$ kubectl -n simple-apps get services`

Output should look like that:

```
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)           AGE
goodbye-service   NodePort    10.0.5.131   <none>        50001:32210/TCP   3s
hello-service     NodePort    10.0.8.13    <none>        50001:32118/TCP   8s
```
  
### Create Ingresses
  
To achieve sticky sessions you will need to create 2 ingress definitions.

Definitions are provided below:

Please change `DOMAIN.NAME` in both ingresses to appropriate to your case. I would advise to look on this Ingress Sticky session link. Both Ingresses are configured to HTTP only traffic.

Apply both of them invoking command:

`$ kubectl apply -f https://raw.githubusercontent.com/kempy007/kube-sticky-session-example/main/nginx/ingresses.yaml`

Check if both configurations were applied:

`$ kubectl -n simple-apps get ingress`

Output should be something like this:

```
NAME              HOSTS        ADDRESS          PORTS   AGE
goodbye-ingress   DOMAIN.NAME   IP_ADDRESS      80      26m
hello-ingress     DOMAIN.NAME   IP_ADDRESS      80      26m
```  

### Test

Open your browser and go to http://DOMAIN.NAME Output should be like this:

```
Hello, world!
Version: 1.0.0
Hostname: hello-549db57dfd-4h8fb
```
  
`Hostname: hello-549db57dfd-4h8fb` is the name of the pod. Refresh it a couple of times.

It should stay the same.

To check if another route is working go to `http://DOMAIN.NAME/v2/` Output should be like this:

```  
Hello, world!
Version: 2.0.0
Hostname: goodbye-7b5798f754-pbkbg
```
  
`Hostname: goodbye-7b5798f754-pbkbg` is the name of the pod. Refresh it a couple of times.

It should stay the same.

To ensure that cookies are not changing open developer tools (probably F12) and navigate to place with cookies. You can reload the page to check if they are not changing.
  
Alternatively you could set the domain name to {IP-ADDRESS}.nip.io

--------
## Traefik

### Apply prerequisites

I followed tutorial REF:https://traefik.io/blog/install-and-configure-traefik-with-helm/

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

I never bothered with the values, just went for a default install with following cmd.

```
helm install traefik traefik/traefik -n traefik --create-namespace
```

### Create ingress & service

Now I found the ingress with annotation on the service did not obey sticky directives. So apply with following cmd

`kubectl apply -f https://raw.githubusercontent.com/kempy007/kube-sticky-session-example/main/traefik/hello-ingressRoute.yaml`

I have left hello-ingress.yaml for reference only.

### test

Find out what the external IP is for the traefik service and browse {IP}.nip.io

`kubectl get svc -A`

You should see something like;

```
Hello, world!
Version: 1.0.0
Hostname: hello-549db57dfd-4h8fb
```


## Appendix

### Versions

DATE: October 2021

AKS Kubernetes: v1.20.9

Nginx Deployment Image;
k8s.gcr.io/ingress-nginx/controller:v1.0.4@sha256:545cff00370f28363dad31e3b59a94ba377854d3a11f18988f5f9e56841ef9ef

Traefik Helm Release;
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
traefik traefik         1               2021-10-20 18:38:14.2237445 +0100 BST   deployed        traefik-10.6.0  2.5.3


### References

- https://stackoverflow.com/questions/59272484/sticky-sessions-on-kubernetes-cluster
- https://jackiechen.blog/2019/11/06/configure-traefik-sticky-session-in-kubernetes/
- https://www.frakkingsweet.com/sticky-sessions-in-traefik-2-1/
- https://traefik.io/blog/install-and-configure-traefik-with-helm/
