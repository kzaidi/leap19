# Create an AKS Cluster

## Select your Subscription

If your Azure identity has access to multiple subscriptions configure the `az` CLI to use your desired subscription for the lab. We will provision an AKS cluster into this subscription in the next step.

```console
$ az account list -o table
```

**Output:**

```console
Name                                            CloudName    SubscriptionId                        State    IsDefault
----------------------------------------------  -----------  ------------------------------------  -------  -----------
Contoso Professional Services                   AzureCloud   25083a1d-6d83-4eb8-b296-51648d940ddf  Enabled  True
My Personal Development Sub                     AzureCloud   57ac26cf-a9f0-4908-b300-9a4e9a0fb205  Enabled  False
```

Set the active Azure subscription via `az account set`:

```console
az account set -s 57ac26cf-a9f0-4908-b300-9a4e9a0fb205
```

## Create Resource Group and AKS Cluster

Create a resource group to hold the cluster. This resource group will be provisioned in West US2.

```console
az group create --name leap19 --location westus2
```

**Output:**

```console
{
  "id": "/subscriptions/57ac26cf-a9f0-4908-b300-9a4e9a0fb205/resourceGroups/foo",
  "location": "westus2",
  "managedBy": null,
  "name": "leap19",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null
}
```

Create a two node AKS cluster running Kubernetes 1.11:

```console
az aks create -g leap19 --name leap19 \
    --kubernetes-version 1.11.5 \
    --node-count 2 \
    --enable-addons http_application_routing \
    --no-ssh-key
```

This cluster create will provision the control plane and attach VMs to your cluster. Initial cluster creation usually takes between 10 and 15 minutes.

If you close your Cloud Shell instance, you can watch for cluster creation to complete:

```console
watch az aks list -g leap19 -o table
```

Remember: Press CTRL+C to stop watching.

**Output:**

```console
Name      Location    ResourceGroup    KubernetesVersion    ProvisioningState    Fqdn
--------  ----------  ---------------  -------------------  -------------------  ------------------------------------------------------
leap19  westus2      leap19         1.11.5               Succeeded            leap19-leap19-57ac26-22b0f7a8.hcp.westus2.azmk8s.io
```

## Fetch cluster credentials

Next, we need to fetch credentials to authenticate to the Kubernetes API. We will ask AKS for the cluster admin credentials so that we can configure Resource Based Access Control (RBAC) in later steps.

```console
az aks get-credentials -g leap19 -n leap19 --admin
```

**Output:**

```console
Merged "leap19-admin" as current context in /Users/jhansen/.kube/config
```

## List cluster nodes

Next, let's talk to the new cluster and list the cluster nodes:

```console
kubectl get nodes
```

Note: you will notice that the only nodes you see are your agent nodes; that's because in this case AKS manages the master nodes for you. You can run this same command using ACS-engine, for example, and you will see all the nodes, master and agent.

**Output:**

```console
NAME                                         STATUS    ROLES     AGE       VERSION
aks-nodepool1-25718494-0                     Ready     agent     1d        v1.11.2
aks-nodepool1-25718494-1                     Ready     agent     1d        v1.11.2
```

## Install Helm

We will use Helm to install software packaged for Kubernetes later in the lab. To quickly install Helm run `./bin/install-helm`:

**Output:**

```console
$ ./bin/install-helm
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
$HELM_HOME has been configured at /Users/jhansen/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```