# Ingress Routing

Ingress routing provides the ability for cluster operators and application developers to describe how traffic should enter the cluster.

The cluster setup in this lab enabled the "HTTP Application Routing" addon. This addon automatically deploys an Ingress stack as part of cluster creation.

The remainder of this lab re-uses that Ingress stack.

## Setup

Let's create a namespace to hold our application and ingress resources:

```console
kubectl create ns ingress
namespace/ingress created
```

```console
kubectl -n ingress get svc,ep,deploy,po
No resources found.
```

As you can see, there are no services, endpoints, or deployments in this new namespace.

## Deploy App-A and App-B

We will deploy two applications to this namespace with their associated services.

```console
kubectl apply -f manifests/03-app-a.yaml
deployment.extensions/app-a created
service/app-a created
```

```console
kubectl apply -f manifests/03-app-b.yaml
deployment.extensions/app-b created
service/app-b created
```

Checking our work, we can see that our applications and services have been deployed to the cluster:

```console
kubectl -n ingress get svc,ep,deploy,po
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/app-a   ClusterIP   10.0.88.67     <none>        80/TCP    6s
service/app-b   ClusterIP   10.0.124.245   <none>        80/TCP    3s

NAME              ENDPOINTS          AGE
endpoints/app-a   10.244.0.24:8080   6s
endpoints/app-b   <none>             3s

NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/app-a   1         1         1            1           7s
deployment.extensions/app-b   1         1         1            1           4s

NAME                         READY     STATUS    RESTARTS   AGE
pod/app-a-85f954584-dnrdf    1/1       Running   0          7s
pod/app-b-845dbd97d9-dsrc6   1/1       Running   0          4s
```

Note that the services included with our applications only expose the apps inside the cluster.

## Accessing our applications

The "HTTP Application Routing" addon automatically created and configured an Azure cloud Load Balancer. To access our applications, we need to discover the Azure public IP address that was allocated. The `kubectl` command may look complicated, but it is using the Service API to extract the IP address assigned to the loadbalancer:

```console
INGRESS_IP=$(kubectl -n kube-system get svc -l app=addon-http-application-routing-nginx-ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
echo $INGRESS_IP
52.183.14.19
```

Let's attempt to connect to our application through this IP address, by using `curl`:

```console
curl -H 'Host: app-a.foo' ${INGRESS_IP}
default backend - 404
```

Instead of finding our application our request was routed to the default backend in our cluster. This resulted in a `404` HTTP response code.

While we have informed our cluster how to run our application, we have not yet created an ingress resource which provides additional routing information.

As with our other resources, apply the ingress configuration:

```console
kubectl apply -f manifests/03-ingress-setup.yaml
ingress.extensions/app-ingress created
```

Checking our work, we can see that `app-a.foo` and `app-b.foo` are now applied to the cluster.

```console
kubectl -n ingress get ingress`
NAME          HOSTS                 ADDRESS   PORTS     AGE
app-ingress   app-a.foo,app-b.foo             80        6s
```

Attempting to access our applications again, we see that traffic for `app-a.foo` and `app-b.foo` are routed to the appropriate pods:

```console
$ curl -H 'Host: app-a.foo' ${INGRESS_IP}
"Service: app-a on app-a-85f954584-dnrdf"
$ curl -H 'Host: app-b.foo' ${INGRESS_IP}
"Service: app-b on app-b-845dbd97d9-dsrc6"
```

Try making up a hostname. Because there is no matching route in our ingress resources, the request will fall through to the default backend.

## Path-based matching

The ingress manifest also included two path-based routing rules. Requests sent to `app-a.foo/b` should be sent to `app-b.foo`:

```console
$ curl -H 'Host: app-a.foo' ${INGRESS_IP}/b
"Service: app-b on app-b-845dbd97d9-dsrc6"
$ curl -H 'Host: app-b.foo' ${INGRESS_IP}/a
"Service: app-a on app-a-85f954584-dnrdf"
```

## Further Reading

[Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/)
[Basic Ingress on AKS](https://docs.microsoft.com/en-us/azure/aks/ingress-basic)
[Enabling TLS Termination with LetsEncrypt](https://docs.microsoft.com/en-us/azure/aks/ingress-tls)

## Cleanup

To cleanup our application and ingress resources simply delete the namespace with `kubectl delete ns ingress`.