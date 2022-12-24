# Machine Learning Model Monitoring 


Machine Learning models are developed, trained and deployed within more and more intelligent applications. Most of the modern applications are developed as cloud-native applications and deployed on Kubernetes.  There are several ways to serve your ML model so that application components can call a prediction REST web service. Seldon Core is a framework that makes it easy to deploy ML models. The Seldon Core metric functionality is an important feature for operational model performance monitoring and can support you observing potential model drift.

In this blog post we are going to explore how to deploy and monitor ML models on OpenShift Kubernetes using Seldon Core, Prometheus and Grafana. 

 

### Topics
* [Install Seldon Core, Prometheus and Grafana](#install-seldon-core-prometheus-and-grafana)
* [Deploy ML models, scrape and graph operational metrics](#deploy-ml-models-scrape-and-graph-operational-metrics)
* [Scrape and graph custom metrics](#scrape-and-graph-custom-metrics)
* [Use the OpenShift 'internal' Prometheus](#use-the-openshift-internal-prometheus)
* [Troubleshooting](#troubleshooting)

### Approach
After installing Seldon Core, Prometheus and Grafana on your OpenShift Kubernetes cluster, we will walk through several basic examples. The machine learning models used here are very simple and just examples for working with metrics.

A basic understanding of OpenShift Kubernetes, Operators, machine learning and git is most likely needed to follow along.
 
Please clone the git repo [openshift-tutorials](https://github.com/stefan-bergstein/openshift-tutorials) on your computer and switch to the directory [ml-monitoring](https://github.com/stefan-bergstein/openshift-tutorials/tree/main/ml-monitoring). 

Additionally, please ensure that you have privileges to deploy and configure operators on your OpenShift cluster. The examples should work fine on [Red Hat CodeReady Containers](https://code-ready.github.io/crc/) as well


# Install Seldon Core, Prometheus and Grafana

Below are step-by-step instructions for installing and configuring Seldon Core, Prometheus and Grafana. Alternatively, you can install these components with [Open Data Hub](http://opendatahub.io). 

Login into OpenShift and create a new project:

```
oc new-project ml-mon
```

## Install the Operators

Install the Operators for Seldon Core, Promentheus and Grafana via the OperatorHub in OpenShift Console or using `oc` CLI and the operator subscriptions in the `operator` directory. E.g.,

```
oc apply -k operator/
```
Sample output:
```
operatorgroup.operators.coreos.com/ml-mon created
subscription.operators.coreos.com/grafana-operator created
subscription.operators.coreos.com/prometheus created
subscription.operators.coreos.com/seldon-operator created

```

In OpenShift Console under `Installed Operators` you should see the following:
![Installed Operators](images/operators.png)


## Create a Prometheus instance and route

Now, create a Prometheus instance and route with by applying the following manifests:

```
oc apply -f prometheus-instance.yaml
oc apply -f prometheus-route.yaml
```

Check if your Prometheus instance is running by navigating in the OpenShift Console to `Networking` -> `Routes`  and clicking on the Prometheus URL. 

## Configure Grafana

Create a Grafana instance:

```
oc apply -f grafana-instance.yaml
```

Your Grafana instance should be connected to you Prometheus instance as datascourde

Therefore, create a Grafana datasource for Prometheus:

```
oc apply -f grafana-prometheus-datasource.yaml
```


Check the route and get the URL for the Grafana dashboard:

```
oc get routes
```
Sample output:

```
NAME              HOST/PORT                                 PATH   SERVICES              PORT      TERMINATION   WILDCARD
grafana-route     grafana-route-ml-mon.apps-crc.testing            grafana-service       3000      edge          None
prometheus        prometheus-ml-mon.apps-crc.testing               prometheus-operated   web                     None
```

Get the URL:

```
echo  http://$(oc get route grafana-route -n ml-mon -o jsonpath='{.spec.host}')
```

Sample output:

```
http://grafana-route-ml-mon.apps-crc.testing
```

In case everything went fine, you should see the **"Welcome to Grafana"** page. 


# Deploy ML models, scrape and graph operational metrics

This this section we will use examples from SeldonIO to deploy ML models, scrape and graph operational metrics.

Ensure you are in the right namespace:
```
oc project ml-mon
```

## Deploy the Seldon Core Grafana Dashboard

The Grafana Operator exposes an API for Dashboards. Let's apply the Prediction Analytics dashboard:

```
oc apply -f prediction-analytics-seldon-core-1.2.2.yaml
```

Open Grafana and have a look the Prediction Analytics dashboard. No data is available yet:

![Empty Prediction Dashboard](images/seldon-dashboard-empty.png)

## Explore Seldon operational metrics

Few steps are needed to see operational metrics:
* Deploy a ML model using Seldon
* Expose the prediction service
* Deploy a Prometheus Service Monitor
* Generate load for the prediction service and view the dashboard

### Deploy a ML model using Seldon

We will use an example from [Seldon Core](https://docs.seldon.io/projects/seldon-core/en/v1.2.2_a/examples/metrics.html#Seldon-Protocol-REST-Model):

```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/release-1.2.2/notebooks/resources/model_seldon_rest.yaml
```

Wait until the pod is deployed:
```
oc get pods
```
Sample output:
```
NAME                                              READY   STATUS    RESTARTS   AGE
...
rest-seldon-model-0-classifier-5594bd9d49-pld7s   2/2     Running   0          91m
...
```

### Expose and test the prediction service

The deployment of the model created a service too.

Expose the created service so that we can test the prediction.
```
oc expose service rest-seldon-model
```

Note, Seldon created two services: ```rest-seldon-model``` and ```rest-seldon-model-classifier```.
We will use here the service ```rest-seldon-model```, because it points to the seldon engine and we have to use the seldon engine to make metrics available for Prometheus. 


Test the prediction service:

```
curl  -H "Content-Type: application/json" -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}' -X POST http://$(oc get route rest-seldon-model  -o jsonpath='{.spec.host}')/api/v1.0/predictions 
```
Sample output:
``` 
{"data":{"names":["proba"],"ndarray":[[0.43782349911420193]]},"meta":{}}
```
The prediction works and the result is 0.437.


### Deploy a Prometheus Service Monitor
Next we will instruct Prometheus to gather Seldon-core metrics for the model. This is done with a Prometheus Service Monitor:

```
oc apply -f rest-seldon-model-servicemonitor.yaml
```

The Service Monitor is going to find the service with the label ```seldon-app=rest-seldon-model``` and scrape metrics from ```/prometheus``` at port ```http```. Here a snippet of the servicemonitor:

```
...
spec:
  endpoints:
  - interval: 30s
    path: /prometheus
    port: http
  selector:
    matchLabels:
      seldon-app: rest-seldon-model
```

### Generate load for the prediction service and view the dashboard

Generate some load for the prediction service to have metric data on the dashboard:
```
while true
do
  curl  -H "Content-Type: application/json" -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}' -X POST http://$(oc get route rest-seldon-model  -o jsonpath='{.spec.host}')/api/v1.0/predictions 
  sleep 2
done

```
The Grafana  Prediction Analytics dashboard will start showing some data:

![Basic Prediction Dashboard](images/seldon-dashboard-basic.png)


## Deploy and monitor a Tensorflow model

Let us repeat the lab with a Tensorflow model from [Seldon Core](https://docs.seldon.io/projects/seldon-core/en/v1.2.2_a/examples/metrics.html#Tensorflow-Protocol-REST-Model):

### Deploy a ML model using Seldon

```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/release-1.2.2/notebooks/resources/model_tfserving_rest.yaml
```

Wait until the pod is deployed:
```
oc get pods
```
Sample output:
```
NAME                                                  READY   STATUS    RESTARTS   AGE
...
rest-tfserving-model-0-halfplustwo-7c6c67fcbc-q6rrk   2/2     Running   0          107s
...
```

### Expose and test the prediction service

Expose the created service so that we can test the prediction.
```
oc expose service rest-tfserving-model
```

Note, Seldon created two services: ```rest-tfserving-model``` and ```rest-tfserving-model-halfplustwo ```.
We will use here the service ```rest-tfserving-model```, because it point to the seldon engine so that we see metrics.


Test the prediction service:

```
curl  -H "Content-Type: application/json" -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://$(oc get route rest-tfserving-model -o jsonpath='{.spec.host}')/v1/models/halfplustwo/:predict
```
Sample output:
``` 
{
    "predictions": [2.5, 3.0, 4.5
    ]
}
```
The prediction works fine! 


### Deploy a Prometheus Service Monitor
Now we will instruct Prometheus to gather Seldon-core metrics for the model. This is done with a Prometheus Service Monitor:

```
oc apply -f rest-tfserving-model-servicemonitor.yaml
```

The Service Monitor is going to find the service with the label ```rest-tfserving-model``` and scrape metrics from ```/prometheus``` at port ```http```. Here a snippet of the servicemonitor::

```
...
spec:
  endpoints:
  - interval: 30s
    path: /prometheus
    port: http
  selector:
    matchLabels:
      seldon-app: rest-tfserving-model
```

### Generate load on the service and view the dashboard

Next, generate some load to see data on the dashboard:
```
while true
do
  curl  -H "Content-Type: application/json" -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://$(oc get route rest-tfserving-model -o jsonpath='{.spec.host}')/v1/models/halfplustwo/:predict 
  sleep 2
done

```
The Grafana Prediction Analytics dashboard will start showing the Tensorflow data. You might have to reload the dashboard.


# Scrape and graph custom metrics

With custom metrics you can expose any metrics from your model. For example you cloud expose features and predictions for model drift monitoring.

Again, let's repeat the lab with a model from [Seldon Core](https://github.com/SeldonIO/seldon-core/blob/v1.2.2/examples/models/custom_metrics/model_rest.yaml).

### Deploy the model and dashboard

Deploy the example model with custom metrics

```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/v1.2.2/examples/models/custom_metrics/model_rest.yaml

```

Deploy a custom dashboard:
```
oc apply -f custom-metrics-dashboard.yaml
```

Open Grafana and have a look the Custom Metrics dashboard. No data is available yet:

![Empty Custom Dashboard](images/custom-dashboard-empty.png)


### Expose the service and test

```
oc expose service seldon-model-example
```

Test the prediction service:

```
curl -s -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}'    -X POST http://$(oc get route seldon-model-example -o jsonpath='{.spec.host}')/api/v1.0/predictions    -H "Content-Type: application/json"

```

Sample output with custom metrics in "meta":
``` 
{"data":{"names":["t:0","t:1","t:2"],"ndarray":[[1.0,2.0,5.0]]},"meta":{"metrics":[{"key":"mycounter","type":"COUNTER","value":1},{"key":"mygauge","type":"GAUGE","value":100},{"key":"mytimer","type":"TIMER","value":20.2}]}}
```

Have a look at the meta data above.

### Scrape custom metrics

**Note,** custom metrics are expose by the predictor (not the engine) at port 6000.

Therefore, add a service for port 6000:

```
oc apply -f seldon-model-example-classifier-metrics-service.yaml 
```

And add a service monitor for the custom metrics:

```
oc apply -f seldon-model-example-classifier-servicemonitor.yaml
```

Create a bit of load:
```
for i in 1 2 3 4 5
do

  curl -s -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}'    -X POST http://$(oc get route seldon-model-example -o jsonpath='{.spec.host}')/api/v1.0/predictions    -H "Content-Type: application/json"
  sleep 1
done
```

The Custom Metrics dashboard will (hopefully) show the data. If not, set the time range of the Grafana dashboard to `Last 15 minutes`.

![Custom Dashboard](images/custom-dashboard-data.png)


So, we saw operational and custom metrics in the Grafana Dashboards. You can now apply these concepts to your ML model serving.


# Use the OpenShift 'internal' Prometheus
In OpenShift Container Platform 4.5, you can enable monitoring for user-defined projects in addition to the default platform monitoring. You can monitor your own projects in OpenShift Container Platform without the need for an additional monitoring solution. Using this new feature centralizes monitoring for core platform components and user-defined projects.

This sections assumes that you have not deployed you own Prometheus Operator. 

## Enabling monitoring for user-defined projects
Follow the steps in [Enabling monitoring for user-defined projects](https://docs.openshift.com/container-platform/4.6/monitoring/enabling-monitoring-for-user-defined-projects.html#enabling-monitoring-for-user-defined-projects_enabling-monitoring-for-user-defined-projects)

The OpenShift Prometheus can now scape metics for you workload including Seldon Core and Seldon custom metric.

## Deploy models and service monitors

Now you can deploy a model and just the service monitor. Let's do this step-by-step for a model using standard and a model with custom metrics:

Ensure you are in the right namespace:
```
oc new-project ml-mon
```

### Deploy a model and view operational metric

#### Deploy the model
```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/release-1.2.2/notebooks/resources/model_seldon_rest.yaml
```

Wait until pods are running and the service is created. 
Next expose the service:

```
oc expose service rest-seldon-model

```
Test the prediction service:

```
curl  -H "Content-Type: application/json" -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}' -X POST http://$(oc get route rest-seldon-model  -o jsonpath='{.spec.host}')/api/v1.0/predictions 
```
Sample output:
``` 
{"data":{"names":["proba"],"ndarray":[[0.43782349911420193]]},"meta":{}}
```
The prediction works and the result is 0.437.


### Deploy a Prometheus Service Monitor
Now we will instruct Prometheus to gather Seldon-core metrics for the model. This is done with a Prometheus Service Monitor:

```
oc apply -f rest-seldon-model-servicemonitor.yaml
```

#### Generate load 

Next, generate some load to have metric data on the dashboard:
```
while true
do
  curl  -H "Content-Type: application/json" -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}' -X POST http://$(oc get route rest-seldon-model  -o jsonpath='{.spec.host}')/api/v1.0/predictions 
  sleep 2
done

```

#### View operational metrics in the OpenShift Developer Console

Now you can open the OpenShift Developer Console and navigate to Monitoring -> Metrics.
Enter a custom query: 
```
round(sum(rate(seldon_api_executor_server_requests_seconds_count{code="200"}[1m]))by (project_name, deployment_name, deployment_version, exported_service, code),0.0001)
``` 
Sample view:
![dev-monitoring-dashboard](images/dev-monitoring-dashboard.png)


### Deploy a model and view custom metric

#### Deploy the model

```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/v1.2.2/examples/models/custom_metrics/model_rest.yaml
```

#### Expose the service
```
oc expose service seldon-model-example
```
Test the prediction service:

```
curl -s -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}'    -X POST http://$(oc get route seldon-model-example -o jsonpath='{.spec.host}')/api/v1.0/predictions    -H "Content-Type: application/json"

```

Sample output with custom metrics in "meta":
``` 
{"data":{"names":["t:0","t:1","t:2"],"ndarray":[[1.0,2.0,5.0]]},"meta":{"metrics":[{"key":"mycounter","type":"COUNTER","value":1},{"key":"mygauge","type":"GAUGE","value":100},{"key":"mytimer","type":"TIMER","value":20.2}]}}
```

#### Scrape custom metrics

**Note,** custom metrics are expose by the predictor (not the engine) at port 6000.

Therefore, add a service for port 6000:

```
oc apply -f seldon-model-example-classifier-metrics-service.yaml 
```

And add a service monitor for the custom metrics:

```
oc apply -f seldon-model-example-classifier-servicemonitor.yaml
```

Create a bit of load:
```
for i in 1 2 3 4 5
do

  curl -s -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}'    -X POST http://$(oc get route seldon-model-example -o jsonpath='{.spec.host}')/api/v1.0/predictions    -H "Content-Type: application/json"
  sleep 1
done
```


#### Custom metrics in the OpenShift Developer Console

Now you can open the OpenShift Developer Console and navigate to Monitoring -> Metrics.
Enter a custom query: 
```
mygauge
``` 
Sample view:
![dev-monitoring-dashboard](images/dev-monitoring-dashboard.png)


# Troubleshooting

## Missing data?
In case your data is not showing up in Grafana, please check first in Prometheus if the metrics and data exist.
![prometheus](images/prometheus.png)


## Internal error occurred: failed calling webhook "v1.mseldondeployment.kb.io":

Deinstalling the Seldon Operator leaves webhookconfiguration behind, which cause trouble when you deploy a seldon deployment for a newly deployed operator,

For example:

```
oc apply -f https://raw.githubusercontent.com/SeldonIO/seldon-core/release-1.2.2/notebooks/resources/model_seldon_rest.yaml
```

Sample output:
```
Error from server (InternalError): error when creating "https://raw.githubusercontent.com/SeldonIO/seldon-core/release-1.2.2/notebooks/resources/model_seldon_rest.yaml": Internal error occurred: failed calling webhook "v1.mseldondeployment.kb.io": Post https://seldon-webhook-service.manuela-ml-workspace.svc:443/mutate-machinelearning-seldon-io-v1-seldondeployment?timeout=30s: service "seldon-webhook-service" not found
```

A previous deployment of the Seldon Operator in ```manuela-ml-workspace``` causes the trouble.

Let's find and delete the WebhookConfiguration. E.g.,
```
oc get MutatingWebhookConfiguration,ValidatingWebhookConfiguration -A | grep manuela
```

Sample output:
```
mutatingwebhookconfiguration.admissionregistration.k8s.io/seldon-mutating-webhook-configuration-manuela-ml-workspace   3          84d
validatingwebhookconfiguration.admissionregistration.k8s.io/seldon-validating-webhook-configuration-manuela-ml-workspace   3          84d
```
Now delete ...
```
oc delete mutatingwebhookconfiguration.admissionregistration.k8s.io/seldon-mutating-webhook-configuration-manuela-ml-workspace
oc delete validatingwebhookconfiguration.admissionregistration.k8s.io/seldon-validating-webhook-configuration-manuela-ml-workspace
```


