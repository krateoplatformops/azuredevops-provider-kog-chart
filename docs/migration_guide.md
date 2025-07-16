# Krateo Azure DevOps Provider KOG - Migration Guide

This document provides a guide for migrating from Krateo Azure DevOps Provider "classic" resources to the new Krateo Azure DevOps Provider KOG resources.

Currently, the Krateo Azure DevOps Provider KOG supports the following resources:
- `GitRepository`
- `Pipeline`
- `PipelinePermission`

Therefore, currently, Krateo Azure DevOps Provider KOG **is not a drop-in replacement** for the Krateo Azure DevOps Provider "classic", but it is a new provider that supports a subset of resources.

## Summary

- [Summary](#summary)
- [Pre-requisites](#pre-requisites)
- [GitRepository migration example](#gitrepository-migration-example)
- [Pipeline migration example](#pipeline-migration-example)
- [PipelinePermission migration example](#pipelinepermission-migration-example)

## Pre-requisites

- You have a the [Krateo Azure DevOps Provider "classic"](https://github.com/krateoplatformops/azuredevops-provider) installed and properly configured in your cluster (including `ConnectorConfig`resource for authentication).
- You have the [Krateo Azure DevOps Provider KOG](https://github.com/krateoplatformops/azuredevops-provider-kog-chart) installed and properly configured in your cluster (including `BasicAuth` resource for authentication).

## `GitRepository` migration example

**Starting point**: `GitRepository` resource managed by Krateo Azure DevOps Provider "classic".
**Ending point**: `GitRepository` resource managed by Krateo Azure DevOps Provider KOG.
Note: the external resource (`GitRepository` on Azure DevOps) will be the same.

Note that the `GitRepository` resource is a non-namespaced resource in the context of Krateo Azure DevOps Provider "classic", while it is a namespaced resource in the context of Krateo Azure DevOps Provider KOG (you can check this by running the following command):
```sh
kubectl api-resources | awk 'NR==1 || $1 == "gitrepositories"'
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
  annotations:
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
  conditions:
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
{{- if and $project $project.status $project.status.id }}
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

**Starting point**: `Pipeline` resource managed by Krateo Azure DevOps Provider "classic".
**Ending point**: `Pipeline` resource managed by Krateo Azure DevOps Provider KOG.
Note: the external resource (`Pipeline` on Azure DevOps) will be the same.

Note that the `Pipeline` resource is a non-namespaced resource in the context of Krateo Azure DevOps Provider "classic", while it is a namespaced resource in the context of Krateo Azure DevOps Provider KOG (you can check this by running the following command):
```sh
kubectl api-resources | awk 'NR==1 || $1 == "pipelines"'
```
Output:
```sh
NAME                                SHORTNAMES   APIVERSION                            NAMESPACED   KIND
pipelines                                        azuredevops.kog.krateo.io/v1alpha1    true         Pipeline
pipelines                                        azuredevops.krateo.io/v1alpha1        false        Pipeline
```

The **starting point** for this migration is the following example of a `Pipeline` resource managed by the Krateo Azure DevOps Provider "classic":
```yaml
apiVersion: azuredevops.krateo.io/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-1
spec:
  name: pipeline-1
  configurationType: yaml
  definitionPath: azure-pipelines.yml
  repositoryType: azureReposGit
  repositoryRef:
    name: repo-1
    namespace: default
  projectRef:
    name: project-1-classic
    namespace: default
  connectorConfigRef:
    namespace: default
    name: connectorconfig-sample
```

Note that the `Pipeline` resource is referecing a `ConnectorConfig` resource, a `Project` resource, and a `GitRepository` resource, which are managed by the Krateo Azure DevOps Provider "classic" and considered pre-requisites for the `Pipeline` resource manged by the Krateo Azure DevOps Provider KOG.

Note that the `GitRepository` referenced in the example above is a `GitRepository` resource managed by the Krateo Azure DevOps Provider "classic". 
However, the `Pipeline` resource managed by the Krateo Azure DevOps Provider KOG will work with both `GitRepository` resources managed by the Krateo Azure DevOps Provider "classic" and the Krateo Azure DevOps Provider KOG since it use the `id` of the repository as the reference.

To ensure that the old version of the resource is not reconciled while you are migrating to the new version, you should set the `krateo.io/paused: true` annotation.
You can do this by running the following commands:
```sh
kubectl annotate pipelines.azuredevops.krateo.io pipeline-1 "krateo.io/paused=true"
```

In addition, in order to keep the external resource (on Azure DevOps) after the deletion of the old `Pipeline` resource on Kubernetes, you need to set `deletionPolicy: Orphan` in the `spec` of the resource.
You can do this with the following command:
```sh
kubectl patch pipelines.azuredevops.krateo.io pipeline-1 \
  --type=merge \
  -p '{"spec":{"deletionPolicy": "Orphan"}}'
```

You can check the changes in the resource with the following command:
```sh
kubectl get pipelines.azuredevops.krateo.io pipeline-1 -o yaml
```

And the output should look like this:
```diff
apiVersion: azuredevops.krateo.io/v1alpha1
kind: Pipeline
metadata:
  annotations:
+   krateo.io/paused: "true"
spec:
  configurationType: yaml
  connectorConfigRef:
    name: connectorconfig-sample
    namespace: default
  definitionPath: azure-pipelines.yml
+ deletionPolicy: Orphan
  name: pipeline-1
  projectRef:
    name: project-1-classic
    namespace: default
  repositoryRef:
    name: repo-1
    namespace: default
  repositoryType: azureReposGit
status:
  conditions:
+  - lastTransitionTime: "2025-07-16T08:33:23Z"
+    reason: ReconcilePaused
+    status: "False"
+    type: Synced
  id: "65"
  url: https://dev.azure.com/krateo-kog/82575162-69b2-4a88-8fd7-bda0d05c0284/_apis/pipelines/65?revision=1
```

Now, you can create a new `Pipeline` resource using the Krateo Azure DevOps Provider KOG following the new schema.
You can find the schema in the specific section of the [README](../README.md#pipeline-schema) file of this chart.
You can apply the following example:
```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-1
  namespace: azuredevops-system                   # Replace with your namespace
  annotations:
    krateo.io/connector-verbose: "true"           # Optional: to enable verbose logging
    krateo.io/deletion-policy: orphan             # Optional: to ensure the external resource is not deleted when the resource is deleted
spec:
  authenticationRefs:
    basicAuthRef: azure-devops-basic-auth         # Reference to a CR containing the basic authentication information.
  
  api-version: "7.2-preview.1"                    # Version of the API to use
  organization: krateo-kog                        # Name of the Azure DevOps organization
  project: "project-1-classic"                    # ID or name of the project
  
  configuration:
    path: azure-pipelines.yml                      # Path to the pipeline configuration file within the repository
    repository: 
      id: "63885da2-bfea-4b13-b19d-cfdb414bccaf"   # ID of the repository where the pipeline is defined
      type: azureReposGit                          # Type of the repository, e.g., gitHub, azureReposGit, etc.
    type: yaml                                     # Type of the pipeline configuration, e.g., yaml, designer, etc.

  name: pipeline-1                                 # Name of the pipeline
EOF
```

The following snippet shows the differences between the old and new `Pipeline` resources:
```diff
- apiVersion: azuredevops.krateo.io/v1alpha1
+ apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-1
+ namespace: azuredevops-system                   # Replace with your namespace, Pipeline is a namespaced resource in the Azure DevOps Provider KOG
+ annotations:
+   krateo.io/connector-verbose: "true"           # Optional: to enable verbose logging
+   krateo.io/deletion-policy: orphan             # Optional: to ensure the external resource is not deleted when the resource is deleted
spec:
  name: pipeline-1
- configurationType: yaml
- definitionPath: azure-pipelines.yml
- repositoryType: azureReposGit
- repositoryRef:
-   name: repo-1
-   namespace: default
- projectRef:
-   name: project-1-classic
-   namespace: default
- connectorConfigRef:
-  namespace: default
-   name: connectorconfig-sample
+ authenticationRefs:
+   basicAuthRef: azure-devops-basic-auth          # Reference to a CR containing the basic authentication information.
  
+ api-version: "7.2-preview.1"                     # Version of the API to use
+ organization: krateo-kog                         # Name of the Azure DevOps organization
+ project: "project-1-classic"                     # ID or name of the project
  
+ configuration:
+   path: azure-pipelines.yml                      # Path to the pipeline configuration file within the repository
+   repository: 
+     id: "58877fa0-7bd2-4f23-959a-7e276d0ee87c"   # ID of the repository where the pipeline is defined
+     type: azureReposGit                          # Type of the repository, e.g., gitHub, azureReposGit, etc.
+   type: yaml                                     # Type of the pipeline configuration, e.g., yaml, designer, etc.
```

Note that the `projectRef` field has been replaced with `project`, which can be either the `ID` or the `name` of the project.
In order to dynamically retrieve the project ID, you can use a `lookup` function.

Note that the `repositoryRef` field has been replaced with `configuration.repository.id`, which is the ID of the repository where the pipeline is defined.

An example of how to use the `lookup` function to retrieve the project ID and repository ID dynamically is shown below.
In this case the context is a Helm chart, so the `lookup` function is used to retrieve the `TeamProject` and `GitRepository` resources by their names and namespace, and then the project ID and repository ID are accessed from the status of those resources.

```yaml
{{- $project := lookup "azuredevops.krateo.io/v1alpha1" "TeamProject" .Release.Namespace (.Values.project.name | lower) }}
{{- if and $project $project.status $project.status.id }}

{{- $repository := lookup "azuredevops.krateo.io/v1alpha1" "GitRepository" .Release.Namespace (.Values.repository.name | lower) }}
{{- if and $repository $repository.status $repository.status.id }}

apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: Pipeline
spec:
  project: "{{ $project.status.id }}"  # Dynamically retrieve the project ID

  configuration:
    repository:
      id: "{{ $repository.status.id }}"  # Dynamically retrieve the repository ID
...

{{- end }}
```

You can check the new `Pipeline` resource managed by Krateo Azure DevOps Provider KOG by running the following command:
```sh
kubectl get pipelines.azuredevops.kog.krateo.io pipeline-1 -n azuredevops-system
```
And the output should look like this:
```sh
NAME         AGE   READY
pipeline-1   86s   True
```

At this point, you can proceed to delete the old `Pipeline` resource managed by Krateo Azure DevOps Provide "classic" (note the different API group).

First, you can delete the old `Pipeline` resource managed by Krateo Azure DevOps Provider "classic":
```sh
kubectl delete pipelines.azuredevops.krateo.io pipeline-1
```

You need also to change the `krateo.io/paused` annotation to `false` to allow the resource to be deleted.
Either you `CTRL+C` the previous command (that is hanging) and run the following command, or you can run it in a separate terminal:
```sh
kubectl annotate pipelines.azuredevops.krateo.io pipeline-1 --overwrite "krateo.io/paused=false"
```


## `PipelinePermission` migration example

**Starting point**: `PipelinePermission` resource managed by Krateo Azure DevOps Provider "classic".
**Ending point**: `PipelinePermission` resource managed by Krateo Azure DevOps Provider KOG.
Note: the external resource (`PipelinePermission` on Azure DevOps) will be the same.

Note that the `PipelinePermission` resource is a non-namespaced resource in the context of Krateo Azure DevOps Provider "classic", while it is a namespaced resource in the context of Krateo Azure DevOps Provider KOG (you can check this by running the following command):
```sh
kubectl api-resources | awk 'NR==1 || $1 == "pipelinepermissions"'
```
Output:
```sh
NAME                                SHORTNAMES   APIVERSION                            NAMESPACED   KIND
pipelinepermissions                              azuredevops.kog.krateo.io/v1alpha1    true         PipelinePermission
pipelinepermissions                              azuredevops.krateo.io/v1alpha2        false        PipelinePermission
```

The **starting point** for this migration is the following example of a `Pipeline` resource managed by the Krateo Azure DevOps Provider "classic":
```yaml
apiVersion: azuredevops.krateo.io/v1alpha2
kind: PipelinePermission
metadata:
  name: pipeline-permission-1
  annotations:
    krateo.io/connector-verbose: "true"
spec:
  authorizeAll: false
  projectRef: 
    name: project-1-classic
    namespace: default
  pipelines:
    - pipelineRef:
        name: pipeline-1
        namespace: default
      authorized: true
  resource:
    type: environment
    resourceRef:
      name: enviroment-1-classic
      namespace: default
  connectorConfigRef:
    namespace: default
    name: connectorconfig-sample
```

Note that the `PipelinePermission` resource is referecing a `ConnectorConfig` resource, a `Environment` resource, and a `Pipeline` resource, which are managed by the Krateo Azure DevOps Provider "classic" and considered pre-requisites for the `PipelinePermission` resource manged by the Krateo Azure DevOps Provider KOG.

Note that the `Pipeline` referenced in the example above is a `Pipeline` resource managed by the Krateo Azure DevOps Provider "classic".  
However, the `PipelinePermission` resource managed by the Krateo Azure DevOps Provider KOG will work with both `Pipeline` resources managed by the Krateo Azure DevOps Provider "classic" and the Krateo Azure DevOps Provider KOG since it use the `id` of the pipeline as the reference.

To ensure that the old version of the resource is not reconciled while you are migrating to the new version, you should set the `krateo.io/paused: true` annotation.
You can do this by running the following commands:
```sh
kubectl annotate pipelinepermissions.azuredevops.krateo.io pipeline-permission-1 "krateo.io/paused=true"
```

In addition, in order to keep the external resource (on Azure DevOps) after the deletion of the old `Pipeline` resource on Kubernetes, you need to set `deletionPolicy: Orphan` in the `spec` of the resource.
You can do this with the following command:
```sh
kubectl patch pipelinepermissions.azuredevops.krateo.io pipeline-permission-1 \
  --type=merge \
  -p '{"spec":{"deletionPolicy": "Orphan"}}'
```

You can check the changes in the resource with the following command:
```sh
kubectl get pipelinepermissions.azuredevops.krateo.io pipeline-permission-1 -o yaml
```

And the output should look like this:
```diff
apiVersion: azuredevops.krateo.io/v1alpha2
kind: PipelinePermission
metadata:
  annotations:
    krateo.io/connector-verbose: "true"
+   krateo.io/paused: "true"
spec:
  authorizeAll: false
  connectorConfigRef:
    name: connectorconfig-sample
    namespace: default
+ deletionPolicy: Orphan
  pipelines:
  - authorized: true
    pipelineRef:
      name: pipeline-1
      namespace: default
  projectRef:
    name: project-1-classic
    namespace: default
  resource:
    resourceRef:
      name: enviroment-1-classic
      namespace: default
    type: environment
status:
  conditions:
+ - lastTransitionTime: "2025-07-16T10:10:59Z"
+   reason: ReconcilePaused
+   status: "False"
+   type: Synced
```

Now, you can create a new `PipelinePermission` resource using the Krateo Azure DevOps Provider KOG following the new schema.
You can find the schema in the specific section of the [README](../README.md#pipelinepermission-schema) file of this chart.
You can apply the following example:
```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: PipelinePermission
metadata:
  name: pipeline-permission-1
  namespace: azuredevops-system                   # Replace with your namespace
  annotations:
    krateo.io/connector-verbose: "true"           # Optional: to enable verbose logging
spec:
  authenticationRefs:
    basicAuthRef: azure-devops-basic-auth         # Reference to a CR containing the basic authentication information.
  
  api-version: "7.2-preview.1"                    # Version of the API to use
  organization: krateo-kog                        # Name of the Azure DevOps organization
  project: "project-1-classic"                    # ID or name of the project
  
  resourceType: "environment"                     # Type of the resource
  resourceId: "35"                                # ID of the resource

  allPipelines:
    authorized: false                             # Whether to authorize all pipelines for the resource

  pipelines:
    - id: 66                                      # ID of the pipeline to authorize for the resource  
      # authorized: true is not required, since it is the default value (authorized: false is not allowed)
EOF
```

The following snippet shows the differences between the old and new `Pipeline` resources:
```diff
- apiVersion: azuredevops.krateo.io/v1alpha2
+ apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: PipelinePermission
metadata:
  name: pipeline-permission-1
  namespace: azuredevops-system                   # Replace with your namespace
  annotations:
    krateo.io/connector-verbose: "true"           # Optional: to enable verbose logging
spec:
  
- authorizeAll: false
- projectRef: 
-   name: project-1-classic
-   namespace: default
- pipelines:
-   - pipelineRef:
-       name: pipeline-1
-       namespace: default
-     authorized: true
- resource:
-   type: environment
-   resourceRef:
-     name: enviroment-1-classic
-     namespace: default
- connectorConfigRef:
-   namespace: default
-   name: connectorconfig-sample
  
+ authenticationRefs:
+   basicAuthRef: azure-devops-basic-auth         # Reference to a CR containing the basic authentication information.
  
+ api-version: "7.2-preview.1"                    # Version of the API to use
+ organization: krateo-kog                        # Name of the Azure DevOps organization
+ project: "project-1-classic"                    # ID or name of the project
  
+ resourceType: "environment"                     # Type of the resource
+ resourceId: "35"                                # ID of the resource

+ allPipelines:
+   authorized: false                             # Whether to authorize all pipelines for the resource

+ pipelines:
+   - id: 66                                      # ID of the pipeline to authorize for the resource  
+     # authorized: true is not required, since it is the default value (authorized: false is not allowed)
```

Note that the `projectRef` field has been replaced with `project`, which can be either the `ID` or the `name` of the project.
In order to dynamically retrieve the project ID, you can use a `lookup` function.

Note that the `resource` field has been replaced with `resourceType` and `resourceId`, which are the type and ID of the resource to authorize for the pipelines. The ID of the resource can be dynamically retrieved using a `lookup` function as well.

Note that the `pipelineRef` fields have been replaced with `id`, which are the ID of the pipelines to authorize for the resource.

An example of how to use the `lookup` function to retrieve the project ID, environment ID and pipeline ID dynamically is shown below.
In this case the context is a Helm chart, so the `lookup` function is used to retrieve the `TeamProject`, `Environment` and `Pipeline` resources by their names and namespace, and then the project ID, environment ID, and pipeline ID are accessed from the status of those resources.

```yaml
{{- $project := lookup "azuredevops.krateo.io/v1alpha1" "TeamProject" .Release.Namespace (.Values.project.name | lower) }}
{{- if and $project $project.status $project.status.id }}

{{- $environment := lookup "azuredevops.krateo.io/v1alpha1" "Environment" .Release.Namespace (.Values.environment.name | lower) }}
{{- if and $environment $environment.status $environment.status.id }}

{{- $pipeline := lookup "azuredevops.krateo.io/v1alpha1" "Pipeline" .Release.Namespace (.Values.pipeline.name | lower) }}
{{- if and $pipeline $pipeline.status $pipeline.status.id }}

apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: PipelinePermission
spec:
  project: "{{ $project.status.id }}"           # Dynamically retrieve the project ID

  resourceType: "environment"                   # Type of the resource
  resourceId: "{{ $environment.status.id }}"    # Dynamically retrieve the environment ID

  pipelines:
    - id: "{{ $pipeline.status.id }}"           # Dynamically retrieve the pipeline ID  
...

{{- end }}
```

You can check the new `PipelinePermission` resource managed by Krateo Azure DevOps Provider KOG by running the following command:
```sh
kubectl get pipelinepermissions.azuredevops.kog.krateo.io pipeline-permission-1 -n azuredevops-system
```
And the output should look like this:
```sh
NAME                    AGE   READY
pipeline-permission-1   61s   True
```

At this point, you can proceed to delete the old `PipelinePermission` resource managed by Krateo Azure DevOps Provide "classic" (note the different API group).

First, you can delete the old `Pipeline` resource managed by Krateo Azure DevOps Provider "classic":
```sh
kubectl delete pipelinepermissions.azuredevops.krateo.io pipeline-permission-1
```

You need also to change the `krateo.io/paused` annotation to `false` to allow the resource to be deleted.
Either you `CTRL+C` the previous command (that is hanging) and run the following command, or you can run it in a separate terminal:
```sh
kubectl annotate pipelinepermissions.azuredevops.krateo.io pipeline-permission-1 --overwrite "krateo.io/paused=false"
```