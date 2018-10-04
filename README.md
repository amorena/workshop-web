# README

This repo contains the source code and configuration files for the Oracle Cloudnative Labs workshop.

Prerequisites:
* an OKE cluster
* a kubeconfig downloaded and loaded
* helm [installed](https://bit.ly/2Qo4XRI)

## Install Prometheus

First, we need to install the Prometheus operator on to our cluster. Follow along with:

[Prometheus Operator on OKE](https://bit.ly/2NbqzPv)

The operator gets installed in the `monitoring` namespace.

```
kubectl get po --namespace monitoring

NAME                                                    READY     STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-0                          2/2       Running   0          5d
factual-greyhound-prometheus-operator-6674d57cb-jr86m   1/1       Running   0          5d
kube-prometheus-exporter-kube-state-66b8849c9b-kgwck    2/2       Running   0          5d
kube-prometheus-exporter-node-6j69h                     1/1       Running   0          5d
kube-prometheus-exporter-node-92v7k                     1/1       Running   0          5d
kube-prometheus-exporter-node-trzsn                     1/1       Running   0          5d
kube-prometheus-grafana-f869c754-xkjmd                  2/2       Running   0          5d
*prometheus-kube-prometheus-0*                            *3/3*       *Running*   *1*          *5d*
```

Let's visit the Prometheus dashboard. *Where is prometheus located?*

```
kubectl port-forward --namespace=monitoring prometheus-kube-prometheus-0 9090:9090
```

We've now port forward prometheus to our localhost, which we can now visit. Open your browser [here](http://localhost:9090)

Investigate **alerts**, **graph** and **status** plus **targets**.

Play around with the following PromQL query.

```
http_request_total
```

## Source code

Let's have a look at our application code. Open up `main.go` in your favorite terminal.

Note that we've imported the Prometheus Go client library:

```
import (
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)
```

And creating the counter

```
var (
	requests = promauto.NewCounter(
		prometheus.CounterOpts{
			Name: "hello_worlds_total",
			Help: "Hello Worlds requests.",
		})
)
```

This adds a basic counter.


Which we increment in our HTTP handler:

```
func handler(w http.ResponseWriter, r *http.Request) {
	requests.Inc()
	w.Write([]byte("Hello, is it me you're looking for?"))
}
```

We've now instrumented our code for prometheus.

## Build with Wercker

build yaml
```
box: golang:1.11
build:
  base-path: /go/src/github.com/mies/workshop-web
  steps:
    - script:
        name: install govendor
        code: go get -u github.com/kardianos/govendor
    
    - script:
        name: install dependencies
        code: govendor sync

    - script:
        name: go vet
        code: govendor vet +local

    # - golint:
    #     exclude: vendor

    - script:
        name: go test
        code: govendor test -v +local

    - script:
        name: go build
        code: |
            ls -la
            go build
            ls -la
```


## Push Docker image to OCIR
push yaml
```
push-release:
  steps:
    - script:
        name: prepare
        code: |
            ls -la /
            ls -la
            #mv workshop-web /workshop-web
            ls -la /

    - internal/docker-push:
        ports: "8080"
        working-dir: /pipeline/source
        cmd: ./workshop-web
        env: "CI=true"
        username: $DOCKER_USERNAME
        password: $DOCKER_PASSWORD
        repository: $DOCKER_REPO
        registry: https://iad.ocir.io/v2
```

## Kubernetes manifest for deployment and monitoring

```
kind: Service
apiVersion: v1
metadata:
  name: workshop-web
  labels:
    app: workshop-web 
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080" 
spec:
  type: NodePort
  selector:
    app: workshop-web
  ports:
  - port: 8080
    targetPort: 8080
    name: http 
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: workshop-web
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: workshop-web
        version: v1
    spec:
      containers:
      - name: workshop-web
        image: iad.ocir.io/oracle-cloudnative/workshop-web
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: workshop-web
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  endpoints:
  - interval: 30s
    port: http
  jobLabel: workshop-web
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: workshop-web
```

Not that we've added a new Object to our deployment manifest called `ServiceMonitor`.

## Manual deploy to OKE

`kubectl create -f app.yaml`

Figure out our pod

```
kubectl get po

kubectl port-forward workshop-web-XXXXXXX-XXXX 8080:8080
```
proxy and curl our app


## Querying Prometheus


```kubectl get svc --namespace monitoring

NAME                                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)             AGE
alertmanager-operated                 ClusterIP      None            <none>          9093/TCP,6783/TCP   5d
kube-prometheus                       NodePort       10.96.19.37     <none>          9090:31941/TCP      5d
kube-prometheus-alertmanager          ClusterIP      10.96.58.27     <none>          9093/TCP            5d
kube-prometheus-exporter-kube-state   ClusterIP      10.96.200.208   <none>          80/TCP              5d
kube-prometheus-exporter-node         ClusterIP      10.96.122.238   <none>          9100/TCP            5d
kube-prometheus-grafana               LoadBalancer   10.96.55.98     XXX.XXX.XXX.X   80:30239/TCP        5d
prometheus-operated                   ClusterIP      None            <none>          9090/TCP            5d
```

## Adding a dashboard to Grafana

Log in with `admin/admin`

Create dashboard
fill in query
show graph
hit url
show graph

## Profit
foo

---
#
# Copyright (c) 2018 Oracle and/or its affiliates. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
