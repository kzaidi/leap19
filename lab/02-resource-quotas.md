# Lab 2 - Resource Quotas

In this lab we will create a namespace, apply resource quotas, and deploy containers into the namespace. Requests that fit within the resource quota will be deployed to the cluster, those that do not will be rejected.

## Create a namespace

First, create a namespace named `restricted` that will hold the resource quota.

```console
kubectl create ns restricted
```

**Output:**

```console
namespace/restricted created
```

To view the newly created namespace, use `kubectl` to list namespaces:

```console
kubectl get ns
```

**Output:**
```
NAME           STATUS    AGE
default        Active     1d
kube-public    Active     1d
kube-system    Active     1d
restricted     Active    40s
```

Note that Kubernetes has a set of default namespaces that are already deployed. For instance, many required cluster services run in `kube-system`.

## Apply resource quotas to the namespace

```console
kubectl apply -f manifests/02-namespace-quota.yaml
```

**Output:**

```console
resourcequota/namespace-quota created
```

You can verify that the quota was applied by using `kubectl describe ns restricted`:

**Output:**

```console
Name:         restricted
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:            namespace-quota
 Resource         Used  Hard
 --------         ---   ---
 limits.cpu       0     2
 limits.memory    0     2Gi
 requests.cpu     0     1
 requests.memory  0     1Gi

No resource limits.
```

In this example, we have provided a hard resource restriction for this namespace which limits total consumption to 1Gi of RAM and 2 CPUs.

## Create an application

Let's create an application in this namespace that conforms to the resource quotas:

```console

kubectl apply -f manifests/02-small-application.yaml

```

**Output:**

```console

pod/small-application created

```

Check the `restricted` namespace to view the pod:

```console

kubectl -n restricted get pod
NAME                READY     STATUS              RESTARTS   AGE
small-application   0/1       ContainerCreating   0          4s

```

After pulling the container image the pod will finish starting and should transition to the `Running` state.

Check current usage in the namespace with `kubectl describe ns restricted`:

**Output:**

```console

Name:         restricted
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:            namespace-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       200m   2
 limits.memory    128Mi  2Gi
 requests.cpu     200m   1
 requests.memory  64Mi   1Gi

 ```

## Create a large application

Next, create an application that requests more resources than are allocated for this namespace:

```console

kubectl apply -f manifests/02-large-application.yaml

```

**Output:**

```console

Error from server (Forbidden): error when creating "manifests/02-large-application.yaml": pods "large-application" is forbidden: exceeded quota: namespace-quota, requested: limits.cpu=2,limits.memory=4Gi,requests.cpu=2,requests.memory=4Gi, used: limits.cpu=200m,limits.memory=128Mi,requests.cpu=200m,requests.memory=64Mi, limited: limits.cpu=2,limits.memory=2Gi,requests.cpu=1,requests.memory=1Gi

```

The request to create the second application pod failed as the resources requested exceed the quota allowed for the namespace.

## Clean up

To remove the resources from this lab, delete the `restricted` namespace:

```console
kubectl delete ns restricted
namespace "restricted" deleted
```

## Diving deeper

For more information on applying default limits to workloads, applying quotas to other resources like secrets and configmaps check out the Kubernetes documentation for [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).