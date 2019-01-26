# Build and deploy a container


## Create an Azure Container Registry instance

ACR instances require a unique name. We will generate a unique suffix and provision ACR into the same resource group as our AKS cluster:
```console
export ACR_NAME=leap19${RANDOM}$$; echo $ACR_NAME

**Output:**
```console
leap19315505330
```

```console
az acr create -g leap19 -n ${ACR_NAME} --sku Standard --location westus2
```

**Output:**
```console
{
  "adminUserEnabled": false,
  "creationDate": "2019-01-25T22:57:05.053546+00:00",
  "id": "/subscriptions/57ac26cf-a9f0-4908-b300-9a4e9a0fb205/resourceGroups/leap19/providers/Microsoft.ContainerRegistry/registries/leap19315505330",
  "location": "westus2",
  "loginServer": "leap19315505330.azurecr.io",
  "name": "leap19315505330",
  "networkRuleSet": null,
  "provisioningState": "Succeeded",
  "resourceGroup": "leap19",
  "sku": {
    "name": "Standard",
    "tier": "Standard"
  },
  "status": null,
  "storageAccount": null,
  "tags": {},
  "type": "Microsoft.ContainerRegistry/registries"
}
```

## Build a container image

```
$ az acr build --registry $ACR_NAME --image helloacrtasks:v1 build
Packing source code into tar to upload...
Uploading archived source code from '/var/folders/tb/3hgy4h655p95lxs3b10g381w0000gp/T/build_archive_498556b23bbc47e1a408790386aeffdc.tar.gz'...
Sending context (2.237 KiB) to registry: leap19315505330...
Queued a build with ID: cc1
Waiting for agent...
2019/01/25 22:57:47 Using acb_vol_7738ab9f-9338-4f6b-bfbd-9b6d85a64228 as the home volume
2019/01/25 22:57:47 Setting up Docker configuration...
2019/01/25 22:57:48 Successfully set up Docker configuration
2019/01/25 22:57:48 Logging in to registry: leap19315505330.azurecr.io
2019/01/25 22:57:49 Successfully logged into leap19315505330.azurecr.io
2019/01/25 22:57:49 Executing step ID: build. Working directory: '', Network: ''
2019/01/25 22:57:49 Obtaining source code and scanning for dependencies...
2019/01/25 22:57:50 Successfully obtained source code and scanned for dependencies
2019/01/25 22:57:50 Launching container with name: build
Sending build context to Docker daemon  12.29kB
Step 1/5 : FROM node:9-alpine
9-alpine: Pulling from library/node
Digest: sha256:8dafc0968fb4d62834d9b826d85a8feecc69bd72cd51723c62c7db67c6dec6fa
Status: Image is up to date for node:9-alpine
 ---> a56170f59699
Step 2/5 : COPY . /src
 ---> 0defe45bab7d
Step 3/5 : RUN cd /src && npm install
 ---> Running in ad847221edca
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN helloworld@1.0.0 No repository field.

up to date in 0.095s
Removing intermediate container ad847221edca
 ---> 3fe073198408
Step 4/5 : EXPOSE 80
 ---> Running in 46a2c6c06626
Removing intermediate container 46a2c6c06626
 ---> 16d3eee98a88
Step 5/5 : CMD ["node", "/src/server.js"]
 ---> Running in 4a6e9794fa01
Removing intermediate container 4a6e9794fa01
 ---> d4bd2f88ec32
Successfully built d4bd2f88ec32
Successfully tagged leap19315505330.azurecr.io/helloacrtasks:v1
2019/01/25 22:57:55 Successfully executed container: build
2019/01/25 22:57:55 Executing step ID: push. Working directory: '', Network: ''
2019/01/25 22:57:55 Pushing image: leap19315505330.azurecr.io/helloacrtasks:v1, attempt 1
The push refers to repository [leap19315505330.azurecr.io/helloacrtasks]
2752b5c89109: Preparing
66aea89e06eb: Preparing
172ed8ca5e43: Preparing
8c9992f4e5dd: Preparing
8dfad2055603: Preparing
66aea89e06eb: Pushed
2752b5c89109: Pushed
8dfad2055603: Pushed
172ed8ca5e43: Pushed
8c9992f4e5dd: Pushed
v1: digest: sha256:2ad92a54b7726ad1e8dec2d65b52a68a336103b2f7b7f53a677e466c0a93e929 size: 1366
2019/01/25 22:58:05 Successfully pushed image: leap19315505330.azurecr.io/helloacrtasks:v1
2019/01/25 22:58:05 Step ID: build marked as successful (elapsed time in seconds: 6.276532)
2019/01/25 22:58:05 Populating digests for step ID: build...
2019/01/25 22:58:07 Successfully populated digests for step ID: build
2019/01/25 22:58:07 Step ID: push marked as successful (elapsed time in seconds: 10.036972)
2019/01/25 22:58:07 The following dependencies were found:
2019/01/25 22:58:07
- image:
    registry: leap19315505330.azurecr.io
    repository: helloacrtasks
    tag: v1
    digest: sha256:2ad92a54b7726ad1e8dec2d65b52a68a336103b2f7b7f53a677e466c0a93e929
  runtime-dependency:
    registry: registry.hub.docker.com
    repository: library/node
    tag: 9-alpine
    digest: sha256:8dafc0968fb4d62834d9b826d85a8feecc69bd72cd51723c62c7db67c6dec6fa
  git: {}

Run ID: cc1 was successful after 23s
```

## Authorize AKS to pull images from ACR

Next, we need to authorize the AKS service principal used by the cluster to access our newly created ACR instance. This requires us to fetch the SP from AKS, the ACR ID, and then add a role assignment:

```console
CLIENT_ID=$(az aks show --resource-group leap19 --name leap19 --query "servicePrincipalProfile.clientId" --output tsv)
ACR_ID=$(az acr show --name $ACR_NAME --resource-group leap19 --query "id" --output tsv)
```

The environment variables `CLIENT_ID` and `ACR_ID` should have something that looks like the following:

```console
$ echo $CLIENT_ID
23129df1-a721-4c06-8ca6-34f6f52a7163
$ echo $ACR_ID
/subscriptions/57ac26cf-a9f0-4908-b300-9a4e9a0fb205/resourceGroups/leap19/providers/Microsoft.ContainerRegistry/registries/leap19315505330
```

Using the `az` CLI, assign the ACR Pull role to your AKS cluster:
```
$ az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID
{
  "canDelegate": null,
  [[ TRIMMED ]]
  "type": "Microsoft.Authorization/roleAssignments"
}
```

## Deploy our application using Helm

Helm is a package manage for Kubernetes. Helm allows application developers to gather the Kubernetes resources for their application into a named, versioned package that can be easily installed, removed, and shared.

```
helm upgrade acr-app --install \
    --namespace default --set image.repository=${ACR_NAME}.azurecr.io/helloacrtasks,image.tag=v1 charts/acr-app/
```

**Output:**
```console
Release "acr-app" does not exist. Installing it now.
NAME:   acr-app
LAST DEPLOYED: Fri Jan 25 15:11:09 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME     TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S)  AGE
acr-app  ClusterIP  10.0.41.224  <none>       80/TCP   0s

==> v1/Deployment
NAME     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
acr-app  1        1        1           0          0s

==> v1/Pod(related)
NAME                      READY  STATUS             RESTARTS  AGE
acr-app-7d9695f989-fcsnz  0/1    ContainerCreating  0         0s
```

Using Helm we can also check the status of our application via `helm status acr-app`:
```
$ helm status acr-app
LAST DEPLOYED: Fri Jan 25 15:11:09 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME     TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S)  AGE
acr-app  ClusterIP  10.0.41.224  <none>       80/TCP   73s

==> v1/Deployment
NAME     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
acr-app  1        1        1           1          73s

==> v1/Pod(related)
NAME                      READY  STATUS   RESTARTS  AGE
acr-app-7d9695f989-fcsnz  1/1    Running  0         73s
```

## Further Reading

[Container Registry Best Practices](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-best-practices)
[Image Quarantine Pattern in ACR (Preview)](https://github.com/Azure/acr/tree/master/docs/preview/quarantine)