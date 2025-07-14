# Azure DevOps Provider KOG - Migration Guide

This document provides a guide for migrating from Krateo Azure DevOps Provider "classic" resources to the new Krateo Azure DevOps Provider KOG resources.

Currently, the Krateo Azure DevOps Provider KOG supports the following resources:
- `GitRepository`
- `Pipeline`
- `PipelinePermission`

## Summary

- [Summary](#summary)
- [Pre-requisites](#pre-requisites)
- [GitRepository migration example](#gitrepository-migration-example)
- [Pipeline migration example](#pipeline-migration-example)
- [PipelinePermission migration example](#pipelinepermission-migration-example)

## Pre-requisites

- You have a the [Krateo Azure DevOps Provider "classic"](https://github.com/krateoplatformops/azuredevops-provider) installed and properly configured in your cluster.
- You have the [Krateo Azure DevOps Provider KOG](https://github.com/krateoplatformops/azuredevops-provider-kog-chart) installed and properly configured in your cluster.

## `GitRepository` migration example

**Starting point**: `GitRepository` resource managed by Krateo Azure DevOps Provider "classic".
**Ending point**: `GitRepository` resource managed by Krateo Azure DevOps Provider KOG.
Note: the external resource (`GitRepository` on Azure DevOps) will be the same.

Note that the `GitRepository` resource is a non-namespaced resource in the context of Krateo Azure DevOps Provider "classic", while it is a namespaced resource in the context of Krateo Azure DevOps Provider KOG (you can check this by running the following command):
```sh
kubectl api-resources | awk 'NR==1 || /git/'
```
Output:
```sh
NAME                                SHORTNAMES   APIVERSION                            NAMESPACED   KIND
gitrepositories                                  azuredevops.kog.krateo.io/v1alpha1    true         GitRepository
gitrepositories                                  azuredevops.krateo.io/v1alpha1        false        GitRepository
```

The **starting point** for this migration is the following example of a `GitRepository` resource managed by the Krateo Azure DevOps Provider "classic":
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

Note that the `GitRepository` resource is referecing a `ConnectorConfig` resource and a `Project` resource, which are both managed by the Krateo Azure DevOps Provider "classic" and considered pre-requisites for the `GitRepository` resource manged by the Krateo Azure DevOps Provider KOG.

To ensure that the old version of the resource is not reconciled while you are migrating to the new version, you should set the `krateo.io/paused: true` annotation.
You can do this by running the following commands:
```sh
kubectl annotate gitrepositories.azuredevops.krateo.io repo-1 "krateo.io/paused=true"
```

In addition, in order to keep the external resource (on Azure DevOps) after the deletion of the old `GitRepository` resource on Kubernetes, you need to set `deletionPolicy: Orphan` in the `spec` of the resource.
You can do this with the following command:
```sh
kubectl patch gitrepositories.azuredevops.krateo.io repo-1 \
  --type=merge \
  -p '{"spec":{"deletionPolicy": "Orphan"}}'
```

You can check the changes in the resource with the following command:
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
+ deletionPolicy: Orphan  
status:
+  - lastTransitionTime: "2025-07-14T15:35:05Z"
+    reason: ReconcilePaused
+    status: "False"
+    type: Synced
```

Now, you can create a new `GitRepository` resource using the Krateo Azure DevOps Provider KOG following the new schema.
You can find the schema in the specific section of the [README](../README.md#gitrepository-schema) file of this chart.
You can apply the following example:
```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: GitRepository
metadata:
  name: repo-1
  namespace: azuredevops-system                   # Replace with your namespace
  annotations:
    krateo.io/connector-verbose: "true"           # Optional: to enable verbose logging
    krateo.io/deletion-policy: orphan             # Optional: to ensure the external resource is not deleted when the resource is deleted
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

An example of how to use the `lookup` function to retrieve the project ID dynamically is shown below.
In this case the context is a Helm chart, so the `lookup` function is used to retrieve the `TeamProject` resource by its name and namespace, and then the project ID is accessed from the status of that resource.

```yaml
{{- $project := lookup "azuredevops.krateo.io/v1alpha1" "TeamProject" .Release.Namespace (.Values.project.name | lower) }}
{{- if and $project $project.status.id }}
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: GitRepository
spec:
  projectId: "{{ $project.status.id }}"  # Dynamically retrieve the project ID
...

{{- end }}
```

You can check the new `GitRepository` resource managed by Krateo Azure DevOps Provider KOG by running the following command:
```sh
kubectl get gitrepositories.azuredevops.kog.krateo.io repo-1 -n azuredevops-system
```
And the output should look like this:
```sh
NAME     AGE   READY
repo-1   10s    True
```

At this point, you can proceed to delete the old `GitRepository` resource managed by Krateo Azure DevOps Provide "classic" (note the different API group).

First, you can delete the old `GitRepository` resource managed by Krateo Azure DevOps Provider "classic":
```sh
kubectl delete gitrepositories.azuredevops.krateo.io repo-1
```

You need also to change the `krateo.io/paused` annotation to `false` to allow the resource to be deleted.
Either you `CTRL+C` the previous command (that is hanging) and run the following command, or you can run it in a separate terminal:
```sh
kubectl annotate gitrepositories.azuredevops.krateo.io repo-1 --overwrite "krateo.io/paused=false"
```

## `Pipeline` migration example

## `PipelinePermission` migration example