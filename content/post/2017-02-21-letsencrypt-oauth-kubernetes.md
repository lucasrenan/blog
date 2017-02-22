---
title:  Let's Encrypt, OAuth 2, and Kubernetes Ingress
date: 2017-02-21
author: Ian Chiles
github: fortytw2
---

In mid-August 2016, fromAtoB switched from running on a few hand-managed bare-metal servers to <a rel="nofollow" href="https://cloud.google.com/">Google Cloud Platform</a> (GCP), using <a rel="nofollow" href="https://saltstack.com/">saltstack</a>, <a rel="nofollow" href="https://www.packer.io/">packer</a>, and <a rel="nofollow" href="https://www.terraform.io/">terraform</a> to
programmatically define and manage our infrastructure. After this migration, it
was relatively straightforward to setup and expose our internal services such as <a rel="nofollow" href="https://www.elastic.co/products/kibana">kibana</a>,
<a rel="nofollow" href="http://grafana.org/">grafana</a>, and <a rel="nofollow" href="https://prometheus.io/">prometheus</a> to the internet at large with a small set of salt states
that managed `oauth2_proxy`, `nginx`, and `lego` on individual machines running
the services managed by systemd.

Then, mid-September, we migrated our Ruby on Rails code to run within Kubernetes
on Google Container Engine (GKE). This was a big change for us on almost all parts of
the stack, as deployments no longer worked with `Capistrano`, but via `kubectl`,
and our CI setup had to change dramatically to allow for docker images to be
built and pushed during CI. However, everything went rather smoothly in the end.

Next, we wanted to migrate our internal services to run within Kubernetes, too -
but we did not have an easy solution to managing Ingress. The Rails application
ran as a `NodePort` service connected to a terraform managed GCP HTTP Load Balancer,
which had all of our main site SSL certificates. Marrying this setup to Let's Encrypt
would not have been very easy, as the GCP HTTP Load Balancer does not support TLS-SNI 
(at the time of this article).

### TLS-SNI and Google Cloud Platform Woes

On GCP, the HTTP load balancers do not support TLS-SNI, which means you need a
new frontend IP address per SSL certificate you have. For internal services, this
is a pain, as you cannot point a wildcard DNS entry to a single IP, like `*.fromatob.com`
and then have everything _just work_. Luckily, we realized that using a TCP
load balancer with the <a rel="nofollow" href="https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx">Nginx IngressController</a>
would work just as well, and support TLS-SNI no problem.

Setting this up was straightforward, by creating a Kubernetes `DaemonSet` that runs
the IngressController on every node, and then pointing a TCP Load Balancer+HealthCheck
to each GKE instance we run.

```yml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx
  namespace: nginx-ingress
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.8.3
        name: nginx
        imagePullPolicy: Always
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        args:
        - /nginx-ingress-controller
        - --default-backend-service=nginx-ingress/default-http-backend
        - --nginx-configmap=nginx-ingress/nginx-ingress-controller
```

### Adding Oauth 2 Authentication

It's relatively important to expose your internal dashboards and services
to the outside world with authentication, and <a rel="nofollow" href="https://github.com/bitly/oauth2_proxy">oauth2 proxy</a> makes this super simple. We like to
run it inside the same Pod that manages our service deployment - for Kibana this
means our deployment looks like

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kibana
  namespace: default
spec:
  replicas: 1
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - image: kibana:5.0.1
        imagePullPolicy: Always
        name: kibana
        env:
        - name: ELASTICSEARCH_URL
          value: "http://elasticsearch:9200"
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 5601
      - name: oauth2-proxy
        image: a5huynh/oauth2_proxy
        args:
          - "-upstream=http://localhost:5601/"
          - "-provider=github"
          - "-cookie-secure=true"
          - "-cookie-expire=168h0m"
          - "-cookie-refresh=60m"
          - "-cookie-secret=SECRET COOKIE"
          - "-cookie-domain=kibana.fromatob.com"
          - "-http-address=0.0.0.0:4180"
          - "-redirect-url=https://kibana.fromatob.com/oauth2/callback"
          - "-github-org=fromAtoB"
          - "-email-domain=*"
          - "-client-id=github oauth ID"
          - "-client-secret=github oauth secret"
        ports:
        - containerPort: 4180
```

and the service for the deployment is just as straightforward -

```yml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 4180
    protocol: TCP
  selector:
    app: kibana
```

### Adding Let's Encrypt

Fortunately for us, integrating Let's Encrypt with Kubernetes via the Nginx Ingress
Controller is easy, thanks to the fantastic <a rel="nofollow" href="https://github.com/jetstack/kube-lego">kube-lego</a> which automatically provisions
SSL certificates for Kubernetes `Ingress` Resources with the addition of a few
simple annotations.

After setting up the appropriate service and deployment for Kibana, simply
creating an `Ingress` resource results in the Nginx being set up and a Let's Encrypt
certificate provisioned for the domain.

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kibana
  namespace: default
  annotations:
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - kibana.fromatob.com
    secretName: kibana-tls
  rules:
  - host: kibana.fromatob.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kibana
          servicePort: 80
```

### Conclusion

With the right setup, it's super easy to expose protected, HTTPS resources from
Kubernetes, if you just want to copy our setup, the manifests we use in production are available on <a rel="nofollow" href="https://github.com/fromatob/manifests">GitHub</a>.

Going forward, we want to investigate setting up <a rel="nofollow" href="https://github.com/zalando-incubator/mate">mate</a> to
automatically provision DNS records from the very same
`Ingress` resources that manage everything else, and switch our main production site to use the same style of
ingress as everything else within Kubernetes.

<hr>

_Want to work with Ian? <a href="mailto:join-our-team@fromatob.com">Say Hi!</a>_
