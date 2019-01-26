# Using Service Accounts

First, create a new namespace `accounting` that will hold our ServiceAccount and application.

```console
kubectl create ns accounting
namespace/accounting created
```

```console
kubectl describe ns accounting

Name:         accounting
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No resource limits.
```

## Creating a service account

Next, create a service account in this namespace named `application-account`:

```console
cat manifests/04-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: application-account
  namespace: accounting
```

```console
kubectl -n accounting apply -f manifests/04-service-account.yaml
serviceaccount/application-account created
```

Kubernetes controllers will automatically generate the necessary authentication token for this service account.

```console
kubectl -n accounting get secrets

NAME                              TYPE                                  DATA      AGE
application-account-token-tdfkb   kubernetes.io/service-account-token   3         15s
default-token-c9s94               kubernetes.io/service-account-token   3         16h
```

## Deploying our application

Next, we will deploy our application into the `accounting` namespace. Our application envokes `kubectl` to list pods. Since we have enabled RBAC on our cluster, but not yet granted any permissions, the token associated with `application-account` will be unable to list pods. Note that the `serviceAccountName` attribute instructs Kubernetes to use the named service account in our application:

```console
cat manifests/04-app.yaml

apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: accounting
spec:
  restartPolicy: Always
  serviceAccountName: application-account
  containers:
    - name: app
      image: lachlanevenson/k8s-kubectl:v1.11.5
      command: [ "/bin/sh", "-c", "kubectl get pods; sleep 30 && date && exit 0"]
      env:
        - name: POD_SLEEP_SECS
          value: "600"
      resources:
        requests:
          memory: "64Mi"
          cpu: "200m"
        limits:
          memory: "128Mi"
          cpu: "200m"

kubectl apply -f manifests/04-app.yaml

deploy/app created
```

Viewing the logs for our application, we can see that the invocation of `kubectl` was rejected:

```console
kubectl -n accounting logs deploy/app
No resources found.
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:accounting:application-account" cannot list pods in the namespace "accounting"
```

## Updating Roles and Bindings

The folling Role instructs Kubernetes to allow the named serivce account `application-account` permission to call the get, watch, and list APIs on pod endpoint:

```console
cat manifests/04-pod-reader-role.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: accounting
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

The following binding associates the `pod-reader` role to subjects inside the cluster. These subjects can be users, groups, or service accounts. Since we have used Roles and RoleBindings the scope of these permissions is limited to the `accounting` namespace:

```console
cat manifests/04-pod-reader-binding.yaml

# This role binding allows "application-account" to read pods in the "accounting" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: accounting
subjects:
- kind: ServiceAccount
  name: application-account
  namespace: accounting
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

Apply the RBAC permissions and force the application to redeploy by deleting the app pod:

```console
kubectl apply -f manifests/04-pod-reader-role.yaml
role.rbac.authorization.k8s.io/pod-reader unchanged

kubectl apply -f manifests/04-pod-reader-binding.yaml
rolebinding.rbac.authorization.k8s.io/read-pods configured
✔  ~/go/src/github.com/slack/leap19
```

```console
kubectl -n accounting delete po --all
pod "app-6fd4db6d5c-t8f9q" deleted
```

View the logs of the new application instance, the service account now has the appropriate permissions to list pods within the cluster:

```console
kubectl -n accounting logs deploy/app
NAME                   READY     STATUS        RESTARTS   AGE
app-6fd4db6d5c-4mzpj   1/1       Running       1          20s
app-6fd4db6d5c-t8f9q   0/1       Terminating   4          2m
```

## Further reading

* [Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
* [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)