---
title: Canary Rollout Kubernetes Objects
---

## Before starting

1. Please make sure you have read the doc of about [deploying helm chart](./helm).

2. Make sure you have already enableld the kruise-rollout addon.
```shell
% vela addon enable kruise-rollout
Addon: kruise-rollout enabled Successfully.
```

3. Please make sure one of the [ingress controllers](https://kubernetes.github.io/ingress-nginx/deploy/) is available in your Kubernetes cluster.
   If not, you can install one in your cluster by enable the [ingress-nginx](../reference/addons/nginx-ingress-controller) addon:

```shell
vela addon enable ingress-nginx
```

Please refer [this](../reference/addons/nginx-ingress-controller) to get the gateway's access address.

**You also can choose to install [traefik](../reference/addons/traefik) addon.**

4. Please make sure the version of  Vela CLI tool `>=1.5.0-alpha.1`, some commands such as rollback rely on the new version.

### First deployment

```shell
cat <<EOF | vela up -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: canary-demo
  namespace: default
  annotations:
    app.oam.dev/publishVersion: v1
spec:
  components:
    - name: canary-demo
      properties:
        objects:
          - apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: canary-demo
            spec:
              replicas: 5
              selector:
                matchLabels:
                  app: demo
              template:
                metadata:
                  labels:
                    app: demo
                spec:
                  containers:
                    - image: barnett/canarydemo:v1
                      name: demo
                      ports:
                        - containerPort: 8090
          - apiVersion: v1
            kind: Service
            metadata:
              labels:
                app: demo
              name: canary-demo
              namespace: default
            spec:
              ports:
                - name: http
                  port: 8090
                  protocol: TCP
                  targetPort: 8090
              selector:
                app: demo
          - apiVersion: networking.k8s.io/v1
            kind: Ingress
            metadata:
              labels:
                app: demo
              name: canary-demo
              namespace: default
            spec:
              ingressClassName: nginx
              rules:
                - host: canary-demo.com
                  http:
                    paths:
                      - backend:
                          service:
                            name: canary-demo
                            port:
                              number: 8090
                        path: /version
                        pathType: ImplementationSpecific
      type: k8s-objects
      traits:
        - type: kruise-rollout
          properties:
            canary:
              steps:
                # The first batch of Canary releases 20% Pods, and 20% traffic imported to the new version, require manual confirmation before subsequent releases are completed
                - weight: 20
                # The second batch of Canary releases 90% Pods, and 90% traffic imported to the new version.
                - weight: 90
              trafficRoutings:
                - type: nginx
EOF
```

Access the gateway endpoint with the specific host.
```shell
$ curl -H "Host: canary-demo.com" <ingress-controller-address>/version
Demo: V1
``` 


## Canary Release
Modify the image tag, from v1 to v2, as follows:

```shell
cat <<EOF | vela up -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: canary-demo
  namespace: default
  annotations:
    app.oam.dev/publishVersion: v2
spec:
  components:
    - name: canary-demo
      properties:
        objects:
          - apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: canary-demo
            spec:
              replicas: 5
              selector:
                matchLabels:
                  app: demo
              template:
                metadata:
                  labels:
                    app: demo
                spec:
                  containers:
                    - image: barnett/canarydemo:v2
                      name: demo
                      ports:
                        - containerPort: 8090
          - apiVersion: v1
            kind: Service
            metadata:
              labels:
                app: demo
              name: canary-demo
              namespace: default
            spec:
              ports:
                - name: http
                  port: 8090
                  protocol: TCP
                  targetPort: 8090
              selector:
                app: demo
          - apiVersion: networking.k8s.io/v1
            kind: Ingress
            metadata:
              labels:
                app: demo
              name: canary-demo
              namespace: default
            spec:
              ingressClassName: nginx
              rules:
                - host: canary-demo.com
                  http:
                    paths:
                      - backend:
                          service:
                            name: canary-demo
                            port:
                              number: 8090
                        path: /version
                        pathType: ImplementationSpecific
      type: k8s-objects
      traits:
        - type: kruise-rollout
          properties:
            canary:
              steps:
                # The first batch of Canary releases 20% Pods, and 20% traffic imported to the new version, require manual confirmation before subsequent releases are completed
                - weight: 20
                # The second batch of Canary releases 90% Pods, and 90% traffic imported to the new version.
                - weight: 90
              trafficRoutings:
                - type: nginx
EOF
```

The configuration strategy of kruise-rollout trait means: The first batch of Canary releases 20% Pods, and 20% traffic imported to the new version, require manual confirmation before subsequent releases are completed.

Check the status of applciation:
```shell
$ vela status canary-demo
About:

  Name:         canary-demo                  
  Namespace:    default                      
  Created at:   2022-06-09 16:43:10 +0800 CST
  Status:       runningWorkflow              

Workflow:

  mode: DAG
  finished: false
  Suspend: false
  Terminated: false
  Steps
  - id:8adxa11wku
    name:canary-demo
    type:apply-component
    phase:running 
    message:wait healthy

Services:

  - Name: canary-demo  
    Cluster: local  Namespace: default
    Type: webservice
    Unhealthy Ready:5/5
    Traits:
      ✅ scaler      ✅ gateway: No loadBalancer found, visiting by using 'vela port-forward canary-demo'
      ❌ kruise-rollout: Rollout is in step(1/1), and you need manually confirm to enter the next step

```

The application's status is `runningWorkflow` that means the application's rollout process has not finished yet.

Access the gateway endpoint again. You will find out there is about 20% chance to meet `Demo: v2` result.

```shell
$ curl -H "Host: canary-demo.com" <ingress-controller-address>/version
Demo: V2
```

## Continue rollout process

After verify the success of the canary version through business-related metrics, such as logs, metrics, and other means, you can resume the workflow to continue the process of rollout.

```shell
vela workflow resume canary-demo
```

Access the gateway endpoint again multi times. You will find out the chance to meet result `Demo: v2` is highly increased, almost 90%.

```shell
$ curl -H "Host: canary-demo.com" <ingress-controller-address>/version
Demo: V2
```

## Canary validation successful, full release

In the end, you can resume again to finish the rollout process.

```shell
vela workflow resume canary-demo
```

Access the gateway endpoint again multi times. You will find out the result always is `Demo: v2`.

```shell
$ curl -H "Host: canary-demo.com" <ingress-controller-address>/version
Demo: V2
```

## Canary verification failed, rollback

If you want to cancel the rollout process and rollback the application to the latest version, after manually check. You can rollback the rollout workflow:

You should suspend the workflow before rollback:
```shell
$ vela workflow suspend canary-demo
Rollout default/canary-demo in cluster  suspended.
Successfully suspend workflow: canary-demo
```

Then rollback:
```shell
$ vela workflow rollback canary-demo
Application spec rollback successfully.
Application status rollback successfully.
Rollout default/canary-demo in cluster  rollback.
Successfully rollback rolloutApplication outdated revision cleaned up.
```

Access the gateway endpoint again. You can see the result always is `Demo: V1`.
```shell
$ curl -H "Host: canary-demo.com" <ingress-controller-address>/version
Demo: V1
```

**Please notice**: rollback operation in middle of a runningWorkflow will rollback to latest succeed revision which is `v1` in this case.
Image a scenario, you deploy a successful `v1` and canary rollout to `v2`, but you find some error in this version, then you continue to rollout to `v3` directly. Error still exists, application is unhealthy then you decide to rollback.
When you execute rollback workload will let application rollback to version `v1`.