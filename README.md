# Working with Google Kubernetes Engine (Autopilot)

This is a follow along step-by-step guide to:
1. Create a GKE Autopilot - Cluster
2. Deploy the Google Microservices Project [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main#readme)
3. Deploy the Datadog Agent via Helm.


## Pre-Requisites:
1. Have access to the [Google Cloud Platform](https://console.cloud.google.com/)
2. Install [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk)
3. Install the [kubectl command-line tool](https://kubernetes.io/docs/tasks/tools/)
## Create a GKE Autopilot Cluster

First we will create a VPC in order to provide network fucntionality to our cluster
```console
gcloud compute networks create NETWORK \
    --subnet-mode=auto \
    --bgp-routing-mode=DYNAMIC_ROUTING_MODE \
    --mtu=MTU
```
Now we can continue by creating our cluster
```console
gcloud container --project "PROJECT_NAME" \
clusters create-auto "CLUSTER_NAME" --region "REGION" \
--release-channel "RELEASE_CHANNEL" \
--network "VPC" \
--subnetwork "SUBNET" \
--cluster-ipv4-cidr "/17"
```
## Deploy the Online Boutique Microservices Project
You can either clone the [original repo](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main#readme) or this one. The main difference is that I have already instrumented instrumented [(Library Injection)](https://docs.datadoghq.com/tracing/trace_collection/library_injection_local/?tab=kubernetes#overview) a few services to report traces to Datadog.
```console
git clone https://github.com/drigo10/Boutique-Microservices.git
```

Deploy the Online Boutique
```console
kubectl apply -f ./release/kubernetes-manifests.yaml
```

Verify that ours pods are in a running state
```console
kubectul get pods
```

To access our frontend in the browser we will add our IP to the allowed load balancer source range

First let's examine our services
```console
kubectl get services
```

```console
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
adservice               ClusterIP      34.118.234.89    <none>         9555/TCP       21h
cartservice             ClusterIP      34.118.238.52    <none>         7070/TCP       21h
checkoutservice         ClusterIP      34.118.236.123   <none>         5050/TCP       21h
currencyservice         ClusterIP      34.118.238.11    <none>         7000/TCP       21h
emailservice            ClusterIP      34.118.238.122   <none>         5000/TCP       21h
frontend                ClusterIP      34.118.229.247   <none>         80/TCP         21h
frontend-external       LoadBalancer   34.118.229.9     34.41.67.108   80:31140/TCP   21h
kubernetes              ClusterIP      34.118.224.1     <none>         443/TCP        21h
paymentservice          ClusterIP      34.118.236.126   <none>         50051/TCP      21h
productcatalogservice   ClusterIP      34.118.227.235   <none>         3550/TCP       21h
recommendationservice   ClusterIP      34.118.229.20    <none>         8080/TCP       21h
redis-cart              ClusterIP      34.118.236.209   <none>         6379/TCP       21h
shippingservice         ClusterIP      34.118.235.63    <none>         50051/TCP      21h
```

Edit the fronend-external of type LoadBalancer
```console
kubectl edit service frontend-external
```

Add your IP to frontend-external Service yaml
```console
  loadBalancerSourceRanges:
  - 255.255.255.255/32
  ```
  Visit your Online Boutique site in a web browser at 
  ```console
  http://EXTERNAL_IP
  ```

## Deploy the Datadog Agent via Helm
Datadog Agent deployment for GKE Autopilot requires additional configuration as noted [here](https://docs.datadoghq.com/containers/kubernetes/distributions/?tab=helm#autopilot).

Specify resource limits for Agent, Trace and Process container. And priority class for the Agent to ensure it is scheduled.

```console
datadog:
  apiKey: <DATADOG_API_KEY>
  appKey: <DATADOG_APP_KEY>
  clusterName: <CLUSTER_NAME>

  # Enable the new `kubernetes_state_core` check.
  kubeStateMetricsCore:
    enabled: true
  # Avoid deploying kube-state-metrics chart.
  # The new `kubernetes_state_core` doesn't require to deploy the kube-state-metrics anymore.
  kubeStateMetricsEnabled: false

agents:
  containers:
    agent:
      # resources for the Agent container
      resources:
        requests:
          cpu: 200m
          memory: 256Mi

    traceAgent:
      # resources for the Trace Agent container
      resources:
        requests:
          cpu: 100m
          memory: 200Mi

    processAgent:
      # resources for the Process Agent container
      resources:
        requests:
          cpu: 100m
          memory: 200Mi

  priorityClassCreate: true

providers:
  gke:
    autopilot: true

```

## Delete Cluster
If cluster is no longer needed, remember to clean up your resources
```console
gcloud container clusters delete CLUSTER-NAME --region REGION
```
## Delete Network Created
```console
gcloud compute networks delete NETWORK_NAME
```










