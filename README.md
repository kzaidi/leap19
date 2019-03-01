# Introduction

This lab walks you through using Aqua Security Platform to scan and protect your containers.

You will explore Kubernetes concepts including resource management, workload isolation, application identity, and container security.

## Lab Setup

For this lab, you will need access to the Aqua Images.   They will be provided to you on the day of the lab. 

## Launch Cloud Shell or use kubectl from your developer machince

Make sure you have access to cloud shell or have kubeclt access from your developer machine


## Checkout the lab material

Next, grab a copy of the materials for this lab by cloning the lab repository from GitHub to your Cloud Shell instance:

You can clone the helm char from here:
https://github.com/aquasecurity/aqua-helm

```console
git clone https://github.com/aquasecurity/aqua-helm

```

## Prerequisites

### Container Registry Credentials

The Aqua server (Console and Gateway) components are available in our private repository, which requires authentication. By default, the charts create a secret based on the values.yaml. 

First, create a new namespace named "aqua":

```bash
kubectl create namespace aqua
```

Next, **(Optional)** create the secret:

```bash
kubectl create secret docker-registry csp-registry-secret  --docker-server="registry.aquasec.com" --namespace aqua --docker-username="jg@example.com" --docker-password="Truckin" --docker-email="jg@example.com"
```

