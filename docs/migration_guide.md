# Azure DevOps Provider KOG Migration Guide

This document provides a guide for migrating from Azure DevOps Provider "classic" resources to the new Azure DevOps Provider KOG resources.

Currently, the Azure DevOps Provider KOG supports the following resources:
- `GitRepository`
- `Pipeline`
- `PipelinePermission`

## Pre-requisites

- You have a the [Azure DevOps Provider "classic"](https://github.com/krateoplatformops/azuredevops-provider) installed and properly configured in your cluster.
- You have the [Azure DevOps Provider KOG](https://github.com/krateoplatformops/azuredevops-provider-kog-chart) installed and properly configured in your cluster.

## `GitRepository` migration

**Starting point**: `GitRepository` resource managed by Azure DevOps Provider "classic".
**Ending point**: `GitRepository` resource managed by Azure DevOps Provider KOG.
Note: the external resource (`GitRepository` on Azure DevOps) will be the same.

Note that the `GitRepository` resource is a non-namespaced resource in the context of Azure DevOps Provider "classic", while it is a namespaced resource in the context of Azure DevOps Provider KOG (you can check this by running the following command):
```sh
kubectl api-resources | awk 'NR==1 || /git/'
```
Output:
```sh
NAME                                SHORTNAMES   APIVERSION                            NAMESPACED   KIND
gitrepositories                                  azuredevops.kog.krateo.io/v1alpha1    true         GitRepository
gitrepositories                                  azuredevops.krateo.io/v1alpha1        false        GitRepository
```

The **starting point** for the migration is the following example of a `GitRepository` resource managed by the Azure DevOps Provider "classic":
```yaml
apiVersion: azuredevops.krateo.io/v1alpha1
kind: GitRepository
metadata:
  name: repo-1
spec:
  connectorConfigRef:
    name: connectorconfig-sample
    namespace: default
  projectRef:
    name: project-1-classic
    namespace: default
  name: repo-1
  initialize: true  
```

Note that the `GitRepository` resource is referecing a `ConnectorConfig` resource and a `Project` resource, which are both managed by the Azure DevOps Provider "classic" and considered pre-requisites for the `GitRepository` resource manged by the Azure DevOps Provider KOG.

In order to keep the external resource (on Azure DevOps) after the deletion of the old `GitRepository` resource on Kubernetes, you need to set the `krateo.io/deletion-policy: orphan` annotation in the metadata of the resource.
In addition, to ensure that the old version of the resource is not reconciled while you are migrating to the new version, you should also set the `krateo.io/paused: true` annotation.
You can do this by running the following commands:
```sh
kubectl annotate gitrepositories.azuredevops.krateo.io repo-1 "krateo.io/deletion-policy=orphan"
kubectl annotate gitrepositories.azuredevops.krateo.io repo-1 "krateo.io/paused=true"
```

You can check the annotations with the following command:
```sh
kubectl get gitrepositories.azuredevops.krateo.io repo-1 -o yaml
```

And the output should look like this:
```diff
apiVersion: azuredevops.krateo.io/v1alpha1
kind: GitRepository
metadata:
  name: repo-1
+ annotations:
+   krateo.io/deletion-policy: orphan
+   krateo.io/paused: "true"
spec:
  connectorConfigRef:
    name: connectorconfig-sample
    namespace: default
  projectRef:
    name: project-1-classic
    namespace: default
  name: repo-1
  initialize: true 
```

Now, you can create a new `GitRepository` resource using the Azure DevOps Provider KOG following the new schema. 
You can apply the following example:
```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: GitRepository
metadata:
  name: repo-1
  namespace: azuredevops-system                # Replace with your namespace
  annotations:
    krateo.io/connector-verbose: "true"
    krateo.io/deletion-policy: orphan
spec:
  authenticationRefs:
    basicAuthRef: azure-devops-basic-auth         # Reference to a CR containing the basic authentication information.
  api-version: "7.2-preview.2"                    # Version of the API to use

  organization: "krateo-kog"                      # Name of the Azure DevOps organization
  projectId: "project-1-classic"                  # ID or name of the project

  name: "repo-1"                                  # Name of the repository to create or manage  
  initialize: true                                # Whether to initialize the repository with a first commit. If set to true, the repository will be initialized with a first commit.
EOF
```

The following snippet shows the differences between the old and new `GitRepository` resources:
```diff
- apiVersion: azuredevops.krateo.io/v1alpha1
+ apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: GitRepository
metadata:
  name: repo-1
+ namespace: azuredevops-system                # Replace with your namespace, GitRepository is a namespaced resource in the Azure DevOps Provider KOG
  annotations:
    krateo.io/connector-verbose: "true"
    krateo.io/deletion-policy: orphan
spec:
- connectorConfigRef:
-   name: connectorconfig-sample
-   namespace: default
- projectRef:
-   name: project-1-classic
-   namespace: default
+ authenticationRefs:
+   basicAuthRef: azure-devops-basic-auth         # Reference to a CR containing the basic authentication information.
+ api-version: "7.2-preview.2"                    # Version of the API to use

+ organization: "krateo-kog"                      # Name of the Azure DevOps organization
+ projectId: "project-1-classic"                  # ID or name of the project

  name: "repo-1"                                  # Name of the repository to create or manage  
  initialize: true                                # Whether to initialize the repository with a first commit. If set to true, the repository will be initialized with a first commit.
```

Note that the `projectRef` field has been replaced with `projectId`, which can be either the `ID` or the `name` of the project.
In order to dynamically retrieve the project ID, you can use a `lookup` function.

At this point, you can proceed to delete the old `GitRepository` resource managed by Azure DevOps Provide "classic" (note the different API group).
First, you need to remove the finalizers from the old resource. You can do this by running the following command:
```sh
kubectl patch gitrepositories.azuredevops.krateo.io repo-1 \
  --type=merge \
  -p '{"metadata":{"finalizers":null}}'
```

Then, you can delete the old resource managed by Azure DevOps Provide "classic" using the following command:
```sh
kubectl delete gitrepositories.azuredevops.krateo.io repo-1
```
