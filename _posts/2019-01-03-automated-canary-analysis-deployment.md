---
layout: post
title: "Automated Canary Analysis Deployment with Istio, Spinnaker on Kubernetes"
author: hoanh
color: rgb(42,140,152)
tags: [tutorial, istio, kubernetes]
---

### 1. Introduction  
<!--more-->
Canary deployment is the way we expose a new version of app to small portion of your application traffic and monitoring the metric related to this version before going ahead the full deployment.
<!--more-->

In basically, canary deployment will compare the behavior of the old and new version of your app. The differences can be subtle and might take some time to appear. You might also have a lot of different metrics to examine.

To solve this problem, [Spinnaker](https://www.spinnaker.io) have an automated canary analysis feature based on Kayenta (a platform for ACA firstly contributed by Netflix). It reads the metrics from both versions from monitoring system and runs a statistical analysis to automate the comparison. This tutorial shows how to do an automated canany analysis on an application deployed on K8s Cluster monitored by Prometheus

### 2. Prerequisites

- A Kubernetes cluster (v 1.15)
- Spinnaker on K8s (click [here](https://medium.com/oracledevs/install-spinnaker-with-halyard-on-kubernetes-88277bd61d59) for installation guideline)
- Istio (click [here](https://truongnh1992.github.io/articles/2019-08/install-istio-bookinfo-app) for installation guide)

### 3. About this tutorial

The app in this tutorial is a simple app whose error rate is configured with an environment variable. Istio uses as traffic management for app. The app expose the metric to Prometheus (Use a component of Istio) and Spinnaker with collects the metrics from Prometheus and make the comparation.

![ACA architecture](https://miro.medium.com/max/2498/1*ssBunShTXwafFg4f4Ph2Gw.png)

### 4. Objective

- Istio gateway deployment
- Expose Prometheus and Grafana service
- Enable Canary feature on Spinnaker
- Add Prometheus as the metric store to spinnaker
- Setup Spinnaker pipeline
- Setup Canary Analysis configuration
- Test the Automated Canary Analysis
- Conclusion

### 5. Istio gateway deployment

In this section, we will deploy an Istio gateway with traffic management function.

5.1. Download the artifacts

```sh
git clone https://github.com/hoabka/automated-cananry-analysis.git
cd automated-cananry-analysis
```

5.2. Apply manifest to create Istio gateway

```sh
kubectl apply -f istio-lb.yaml
```

5.3. Verify gateway

```sh
kubectl get gateway
```

In the manifest, it will create a http gateway named “**sample-gateway**”. It is ingressgateway type and listen on port 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: sampleapp-gateway
  labels:
    app: sampleapp-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

Next code spinets, it will create Routing policy

```yaml
http:
  - route:
    - destination:
        host: sampleapp
        subset: v1
      weight: 80
    - destination:
        host: sampleapp
        subset: baseline
      weight: 10
    - destination:
        host: sampleapp
        subset: canary
      weight: 10
```

It will route client traffic to 3 subsets (v1, baseline and canary) with correspond rate. In our tutorial, 80% request will be routed to “prod” pods and 10% for baseline and canary.

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: sampleapp
spec:
  host: sampleapp
  subsets:
  - name: v1
    labels:
      version: prod
  - name: baseline
    labels:
      version: sampleapp-baseline
  - name: canary
    labels:
      version: sampleapp-canary
```

In the last code spinet, we define the **DestinationRule** which used to categorize the subset based on the pod label.

5.4. Edit gateway svc as NodePort type (Some CloudProvider doesn’t allow LoadBalancer type)

```sh
kubectl edit svc istio-ingressgateway -n istio-system
```

Change type “**Loadbalancer**” => “**NodePort**”

```yaml
selector:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
  sessionAffinity: None
  type: NodePort
```

Verify the svc

```sh
kubectl get svc -n istio-system -l app=istio-ingressgateway
```

Result (don’t run cmd)

```console
#Result
istio-ingressgateway   NodePort   10.99.81.161   <none>        15020:32481/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:30815/TCP,15030:30822/TCP,15031:31448/TCP,15032:31589/TCP,15443:32280/TCP   5d20h
```

Our gateway application will be exposed on port **31380**.

Later on, you can access to apps by visiting [http://node_ip:31380](http://node_ip:31380)

### 6. Expose Prometheus and Grafana service

In this part, we will expose the Prometheus and Grafana service. These service include to Istio component which used to monitor Istio.

6.1. Expose Prometheus and Grafana service:

```yaml
cat << EOF | kubectl apply -f
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  labels:
    run: grafana-service
  namespace: istio-system
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    protocol: TCP
    nodePort: 31110
    name: http
  selector:
    app: grafana
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  labels:
    run: prometheus-service
  namespace: istio-system
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
    nodePort: 31100
    name: http
  selector:
    app: prometheus
EOF
```

From above manifest, we expose Grafana service on NodePort 31110 and Prometheus on 31100 base on the pod label in istio-system namespace.

6.2. Access to Prometheus and Grafana

You can acces to Prometheus UI by visting: [http://node_ip:31100](http://node_ip:31100) and

![prometheus](https://miro.medium.com/max/3780/1*pRrKo1W_ec9fTAhMq4nB9Q.png)

Grafana on [http://node_ip:31110](http://node_ip:31110) (default pw: admin/admin)

![grafana](https://miro.medium.com/max/3836/1*OEvdx1wvYJhEBr2GF7inQA.png)

### 7. Enable Canary feature on Spinnaker

From this [tutorial](https://medium.com/oracledevs/install-spinnaker-with-halyard-on-kubernetes-88277bd61d59). You login into **halyard** container and do later action below.

```
hal config canary enable
hal config canary prometheus enable
hal config canary prometheus account add my-prometheus --base-url http://node_ip:31100
```

We enable canary config to make canary feature available on UI and then add prometheus as the source metric.  

The prometheus server base url get from above part: [http://node_ip:31100](http://node_ip:31100)

```
hal config canary edit --default-metrics-store prometheus
hal config canary edit --default-metrics-account my-prometheus
```

Then we add default metric store and default metric account.

```
hal deploy apply
```

Reflex the changes

![canary](https://miro.medium.com/max/3840/1*jMoZ52fwzaAY-NYjs0LZew.png)

And lastly, move to Spinnaker UI ==> CONFIG ==> FEATURES ==> Tick on Canary.

### 8. Setup Spinnaker pipeline

We will create 2 pipelines for our tutorial

* **prd-deploy** for production deployment (just do it, don’t be serious :D). It will be run for the very first time and will be triggered from **automated-canary-analysis** pipeline later on

* **automated-canary-analysis** for canary deployment and analysis

Create pipelines

![pipeline](https://miro.medium.com/max/3852/1*n_QoqJzJllUEenXgz32O8w.png)

Pipeline configuration

![config](https://miro.medium.com/max/3840/1*kENvvC-zHylAc13jOhs88A.png)

Copy the content in file **prd-deploy.json** into JSON editor on prd-deploy pipeline and **automated-canary-analysis.json** as well.  

After complete to edit JSON. You will look 2 pipelines as below.

![prd-deploy](https://miro.medium.com/max/2836/1*IknOE-VEcefWHI88vfXtWw.png)

![automate-canary-analysis](https://miro.medium.com/max/2826/1*cU2Vi6B8xzlMRMp3Jwjlaw.png)

Edit the K8s account in every stage with your K8s account.

![k8s_account](https://miro.medium.com/max/3350/1*JEgjxqd8sUvYQFTEI_tHQA.png)

> Your K8s account is enabled already in this [tutorial](https://medium.com/oracledevs/install-spinnaker-with-halyard-on-kubernetes-88277bd61d59)

### 9. Setup Canary configuration

![canary_config](https://miro.medium.com/max/2746/1*u2rLrMIMY4fivjfwicNrsA.png)

In the canary config, we created new config named: **canary-analysis**, add new Group metric named **API REQUEST** with metric name **Error5xx and Latency**

> Note: you can create multiple groups of metrics to get more detailed about the application by canary analysis. In the scope of tutorial, to make it simple, I just created only one group with 2 metrics

In SCORING part, we rate the score for Group metrics. You can define what is more important group metric by rating with higher score (higher piority). The total score is 100.

> Example: API REQUEST: 60, Resource ultilization” 40

![Error5xx](https://miro.medium.com/max/1252/1*Ybzo5zSV9cpFy-3y1DyquA.png)

Let’s query the metric in Prometheus web view to get its labels:

![query](https://miro.medium.com/max/3812/1*PtKH41Oj2Bspt7DG09GcVg.png)

For the configuration we need some labels to differentiate the baseline and canary instances. As you can see by looking at the name-value pairs for each instance, we can use combination of “**destination_version**” and “**destination_workload_namespace**” to identify the instance of our application. Let’s take a look at the fields that we have in the **Metric Scope** of the **Canary Analysis** stage:

![canary_pair](https://miro.medium.com/max/2204/1*xI8t6arfUh_3w6sesanwtg.png)

Prometheus query to calculate average 5xx errors on 1 minute.

```
rate(istio_requests_total{connection_security_policy="unknown", response_code="500", destination_version = "${scope}", destination_workload_namespace="${location}"}[1m])
```

Query to get latency

```
sum(irate(istio_request_duration_seconds_bucket{reporter="destination", connection_security_policy!="mutual_tls", destination_service=~"sampleapp.default.svc.cluster.local", destination_workload="${scope}", destination_workload_namespace="${location}"}[1m]))
```

### 10. Test the Automated Canary Analysis

Run the **prd-deploy pipeline** first with **SUCCESS_RATE=90**. That means, we expect our application will response to client requests with 10% error (90% success).

prd-deploy with SUCCESS_RATE=90
![prd-deploy](https://miro.medium.com/max/1202/1*rPHCAIqQjTJ80Upj_LMYmw.png)

Deployment successfully
![success](https://miro.medium.com/max/2176/1*E4rYkKvyJs0LzpZhluC6mQ.png)

Create client traffic to our istio gateway

```
while true; do curl http://node_ip:31380; done
```

After **prd-deploy pipeline** completed. Click to run **automated-canary-analysis pipeline SUCCESS_RATE=70**.
![confirm](https://miro.medium.com/max/1202/1*bmXFQPtpjfNKPJslOSsq0A.png)

This **SUCCESS_RATE** parameter is used for canary version.  

That means, in our tutorial, we will setup the baseline version with **SUCCESS_RATE=90** (a portion of current running version) and canary version with **SUCCESS_RATE=70** (error rate 5xx will be 30%)  

Canary Analysis will compare the total 5xx responses from both baseline and canary version and then giving the results.

Canary summary
![summary](https://miro.medium.com/max/1740/1*i1XOERpvG-9oxDCUk2dOGQ.png)

Detailed 5xx with baseline and canary
![5xx](https://miro.medium.com/max/2766/1*VP4Sxqq14npgbgJgAabRIw.png)

Grafana dashboard
![dash](https://miro.medium.com/max/3786/1*SAvL4k-9RAVS5fUPgn3Z2Q.png)

Because, we setup SUCCESS_RATE=90 in **prd-deploy** pipeline, so the baseline will be same SUCCESS_RATE, so there is 10% error5xx in baseline version.  

For further testing, set **SUCCESS_RATE=90** for both baseline (from **prd-deploy** pipeline) and canary. The canary analysis will be successfully.

![successfully](https://miro.medium.com/max/1482/1*dWMdzwbVjmt3bneLHCxG6w.png)


### Conclusion

- By using Automated Canary Analysis, we can save time to analyze the behavior of new version
- Let think about the important metrics you wanna configure for Canary Analysis and set the score for them. More important, higher score
- Long time analysis, more confidence in the final canary score.  


**Happy hacking :)**
