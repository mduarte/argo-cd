# Ingress Configuration

ArgoCD runs both a gRPC server (used by the CLI), as well as a HTTP/HTTPS server (used by the UI).
Both protocols are exposed by the argocd-service on the following ports:
* 443 - gRPC/HTTPS
* 80 - HTTP (redirects to HTTPS)

There are several ways how Ingress can be configured.

## [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

### Option 1: ssl-passthrough

Because multiple protocols (gRPC/HTTPS) are being served on the same port (443), this provides a
challenge when attempting to define a single nginx ingress object and rule for the argocd-service,
since the `nginx.ingress.kubernetes.io/backend-protocol` [annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-protocol)
accepts only a single value for the backend protocol (e.g. HTTP, HTTPS, GRPC, GRPCS).

In order to expose the ArgoCD API server with a single ingress rule and hostname, the
`nginx.ingress.kubernetes.io/ssl-passthrough` [annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#ssl-passthrough)
must be used to passthrough TLS connections and terminate TLS at the ArgoCD API server.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
```

The above rule terminates TLS at the ArgoCD API server, which detects the protocol being used,
and responds appropriately. Note that the `nginx.ingress.kubernetes.io/ssl-passthrough` annotation
requires that the `--enable-ssl-passthrough` flag be added to the command line arguments to
`nginx-ingress-controller`.

### Option 2: Multiple ingress objects and hosts

Since ingress-nginx Ingress supports pm;u a single protocol per Ingress object, an alternative
way would be to define two Ingress objects. One for HTTP/HTTPS, and the other for gRPC:

HTTP/HTTPS Ingress:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: http
    host: argocd.example.com
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret
```

gRPC Ingress:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
    host: grpc.argocd.example.com
  tls:
  - hosts:
    - grpc.argocd.example.com
    secretName: argocd-secret
```

The API server should then be run with TLS disabled using the `--insecure` flag.

The obvious disadvantage to this approach is that this technique require two separate hostnames for
the API server -- one for gRPC and the other for HTTP/HTTPS. However it allow TLS termination to
happen at the ingress controller.


## AWS Application Load Balancers (ALBs) and Classic ELB (HTTP mode)

Neither ALBs and Classic ELB in HTTP mode, do not have full support for HTTP2/gRPC which is the
protocol used by the `argocd` CLI. Thus, when using an AWS load balancer, either Classic ELB in 
passthrough mode is needed, or NLBs.