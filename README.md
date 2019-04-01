# google-analytics-proxy

Some users use ad blockers (or other network filters) which will block Google Analytics scripts and requests. Inaccurate data is bad. This repository offers a tiny Nginx proxy for Google Analytics and Google Tag Manager to get metrics for people with blockers.

The proxy carefully forwards the client real IP address to Analytics using the [Google Analytics Measurement Protocol](https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters#uip) URL parameter. So you don't have to worry seeing the proxy's IP instead of client's in GA.

## Usage
1. Have a look at `Dockerfile` and `nginx.conf`. Basically you will only need to override `GAPROXY_HOST` environment variable to get the proxy work.
2. If this proxy is placed behind another reverse proxy, add `X-Real-IP` header in the front reverse proxy so `google-analytics-proxy` can forward the real client IP to Google Analytics. 
3. Replace `gtag.js`, `analytics.js`, `gtm.js`, `ns.html` with the one under your proxy domain.

### Deploy in Kubernetes cluster

`01deployment.yaml`:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: google-analytics-proxy
  namespace: default
  labels:
    app: google-analytics-proxy
spec:
  replicas: 1
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: google-analytics-proxy
    spec:
      containers:
      - name: proxy
        image: fr0der1c/google-analytics-proxy:latest
        env:
        - name: "GAPROXY_HOST"
          value: "ga.my-domain.com"
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
          limits:
            cpu: 200m
            memory: 100Mi
        imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30

```

`02service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: google-analytics-proxy
  namespace: default
  labels:
    app: google-analytics-proxy
spec:
  ports:
    - name: "tcp-80-80"
      protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app: google-analytics-proxy
  sessionAffinity: None
  externalTrafficPolicy: "Cluster"
  type: LoadBalancer
```