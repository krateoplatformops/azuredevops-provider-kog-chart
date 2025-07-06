# Azure DevOps Provider KOG Helm Chart

This is a [Helm Chart](https://helm.sh/docs/topics/charts/) that deploys the Krateo Azure DevOps Provider KOG leveraging the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider).
This provider allows you to manage [Azure DevOps resources](https://azure.microsoft.com/en-us/products/devops) such as `pipelinepermissions`, `pipelines` and `gitrepositories` using the Krateo platform.

> [!NOTE]  
> This chart is still in development and not yet ready for production use.

## Summary

- [Summary](#summary)
- [Requirements](#requirements)
- [How to install](#how-to-install)
- [Use in parallel with Azure DevOps Provider (classic)](#use-in-parallel-with-azure-devops-provider-classic)
- [Supported resources](#supported-resources)
  - [Resource details](#resource-details)
    - [PipelinePermission](#pipelinepermission)
      - [Operations](#operations)
      - [Example](#example)
      - [How to revoke permissions](#how-to-revoke-permissions)
      - [Changes in the OpenAPI Specification](#changes-in-the-openapi-specification)
- [Authentication](#authentication)
- [Configuration](#configuration)
  - [values.yaml](#valuesyaml)
  - [Verbose logging](#verbose-logging)
- [Chart structure](#chart-structure)
- [Troubleshooting](#troubleshooting)

## Requirements

[Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider) should be installed in your cluster. Follow the related Helm Chart [README](https://github.com/krateoplatformops/oasgen-provider-chart) for installation instructions.

## How to install

To install the chart, use the following commands:

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install azuredevops-provider-kog krateo/azuredevops-provider-kog
```

> [!NOTE]
> Due to the nature of the providers leveraging the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider), this chart will install a set of RestDefinitions that will in turn trigger the deployment of controllers in the cluster. These controllers need to be up and running before you can create or manage resources using the Custom Resources (CRs) defined by this provider. This may take a few minutes after the chart is installed.

You can check the status of the controllers by running:
```sh
until kubectl get deployment azuredevops-provider-kog-<RESOURCE>-controller -n <YOUR_NAMESPACE> &>/dev/null; do
  echo "Waiting for <RESOURCE> controller deployment to be created..."
  sleep 5
done
kubectl wait deployments azuredevops-provider-kog-<RESOURCE>-controller --for condition=Available=True --namespace <YOUR_NAMESPACE> --timeout=300s
```

Make sure to replace `<RESOURCE>` to one of the resources supported by the chart, such as `pipelinepermission`, `pipeline`, `gitrepository`, and `<YOUR_NAMESPACE>` with the namespace where you installed the chart.

## Use "in parallel" with Azure DevOps Provider (classic)

This chart can be used in parallel with the [Azure DevOps Provider (classic)](https://github.com/krateoplatformops/azuredevops-provider).
As a matter of fact, currently, this chart allows you to manage the following resources:
- `PipelinePermission`

Other resources (`TeamProject`, `Queue`, `Environment`, etc.) can be managed using the [Azure DevOps Provider (classic)](https://github.com/krateoplatformops/azuredevops-provider) and referenced by the resources managed by this chart.
For example, you can create a `PipelinePermission` resource that references a `Pipeline` resource created by the Azure DevOps Provider (classic).
> [!NOTE]  
> These references are "by id" or other Azure DevOps resource identifiers but not Kubernetes-native. Meaning that the `PipelinePermission` resource will reference the Azure DevOps pipeline by its `id`, not by a Kubernetes resource name and namespace. Said `id` can be found in the Status field of the `Pipeline` resource created by the Azure DevOps Provider (classic).

## Supported resources

This chart supports the following resources and operations:

| Resource           | Get  | Create | Update | Delete |
|--------------------|------|--------|--------|--------|
| PipelinePermission | ✅   | ✅     | 🟡     | 🚫 Not supported    |
| GitRepository      | ✅   | ✅     | 🟡     | ✅     |

> [!NOTE]  
> 🚫 *"Not supported"* means that the operation is not supported by the resource (e.g., the underlying REST API does not support it and therefore the controller does not implement it) while 🚫 *"Not applicable"* means that the operation does not apply to the resource.

> [!NOTE]  
> 🟡 *"Partial"* means that the operation is only partially supported — for example, only some fields are implemented.

The resources listed above are Custom Resources (CRs) defined in the `azuredevops.kog.krateo.io` API group. They are used to manage Azure DevOps resources in a Kubernetes-native way, allowing you to create, update, and delete Azure DevOps resources using Kubernetes manifests.

### Resource examples

You can find example resources for each supported resource type in the `/samples` folder of the chart.
These examples Custom Resources (CRs) show every possible field that can be set in the resource based reflected on the Custom Resource Definitions (CRDs) that are generated and installed in the cluster.

#### PipelinePermission

The `PipelinePermission` resource is used to manage permissions for Azure DevOps pipelines.

##### Operations

- **Create**: You can create a `PipelinePermission` resource to grant permissions for specific pipelines to use a specific resource, such as `environment`, `queue`, etc.
- **Update**: Updating is only partially supported, meaning that you cannot directly revoke permissions (change the `authorized` field to `false`) for existing pipelines (see the [section below](#how-to-revoke-permissions) on how to revoke permissions for pipelines). You can only add new pipelines with `authorized: true`. You can also change permissions for all pipelines in the project by setting the `allPipelines` field to `authorized: true` or `authorized: false`.
- **Delete**: Deleting a `PipelinePermission` custom resource will not revoke permissions for the pipelines, it will only remove the resource from the cluster and the controller will stop managing it. The Azure DevOps pipelines will still have the permissions granted by the `PipelinePermission` resource until you manually revoke them in the Azure DevOps UI.

##### Example

An example of a PipelinePermission resource is:
```yaml
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: PipelinePermission
metadata:
  name: test-pp
  namespace: adp
  annotations:
    krateo.io/connector-verbose: "true"
spec:
  authenticationRefs:
    basicAuthRef: azure-devops-basic-auth
  api-version: 7.2-preview.1
  
  organization: "krateo-kog"
  project: "test-project-1-classic"
  resourceType: "environment" 
  resourceId: "7"

  allPipelines:
    authorized: false

  pipelines:
    - id: 14
      # authorized: true is not required, since it is the default value (authorized: false is not allowed)
    - id: 15
```

##### How to revoke permissions

To revoke permissions for a pipeline, you need to:
1. Manually remove the specific `Pipeline` in the Azure DevOps UI under the `Pipeline permission` section of the resource you want to manage (e.g., `Environment`, `Queue`, etc.).
2. Update the `PipelinePermission` resource by removing the specific pipeline from the `pipelines` array in the `PipelinePermission` resource. 

##### Main changes w.r.t. original OpenAPI Specification

Since the Azure DevOps REST API returns only the pipelines that are authorized for the user, the `PipelinePermission` resource allows you to set the `authorized` field of each `pipeline` in the `pipelines` array to `true` only.
Therefore, the OpenAPI Specification (OAS) of the `PipelinePermission` resource has been modified to restrict the `authorized` field to only accept `true` and set it as the default value.
This is done by defining a custom schema named `PermissionTrueOnly`.
In addition, the `ResourcePipelinePermissionsTrueOnly` and `PipelinePermissionTrueOnly` schemas are defined.
This is done to address the PATCH operation defined in the OAS.
The PATCH operation is used in the RestDefinition `pipelinepermission` for the `create` and `update` operations.

```diff
- Permission:
+ PermissionTrueOnly:
    type: object
    properties:
      authorized:
        type: boolean
+       enum:
+       - true          # Only true allowed in the CR
+       default: true   # Default value is true
```

#### GitRepository

##### Fork-related fields

https://learn.microsoft.com/en-us/rest/api/azure/devops/git/repositories/create?view=azure-devops-rest-7.2&tabs=HTTP#create-a-fork-of-a-parent-repository

##### Main changes w.r.t. original OpenAPI Specification


changed id in path of the GET /{organization}/{project}/_apis/git/repositories/{repositoryId}
`repositoryId` to `id`

---
changed `project` to `projectId` for the path parameter.

otherwise there is a
clash between project in path and project in response body

note that projectId could be either a project name or a project ID,

---


In the 
delete endpoint

/{organization}/{projectId}/_apis/git/repositories/{id}

changed from 200 to 
      responses:
        "204":

as 204 is the status code that is actually returned by the Azure DevOps REST API when a repository is deleted successfully.

---
added new schemas

schema `GitRepositoryUpdateOptions` to allow updating the name and default branch of a Git repository.
therfore only a subset of the original


minimal schemas
















## Authentication

The authentication to the Azure DevOps REST API is managed using 2 resources (both are required):

- **Kubernetes Secret**: This resource is used to store the Azure DevOps Personal Access Token (PAT) that is used to authenticate with the Azure DevOps REST API. The PAT should have the necessary permissions to manage the resources you want to create, update or delete.

In order to generate a Azure DevOps token you could follow the steps described in the [Azure DevOps documentation](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate).

Example of a Kubernetes Secret that you can apply to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: azuredevops-secret
  namespace: krateo-system
type: Opaque
stringData:
  token: <PAT>
EOF
```

Replace `<PAT>` with your actual Azure DevOps Personal Access Token.

- **BasicAuth**: This resource references the Kubernetes Secret and is used to authenticate with the Azure DevOps REST API. Then, it is used in the `authenticationRefs` field of the Custom Resources (like `PipelinePermission`, `Pipeline`, and `GitRepository`) defined in this chart.

Example of a BasicAuth resource that references the Kubernetes Secret, to be applied to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: BasicAuth
metadata:
  name: azure-devops-basic-auth
  namespace: adp
spec:
  username: "anything"  # Any value as official Azure DevOps OAS suggests (field not used)
  passwordRef:
    name: azuredevops-secret
    namespace: default
    key: token
EOF
```

## Configuration

### values.yaml

You can customize the chart by modifying the `values.yaml` file.
For instance, you can select which resources the provider should support in the oncoming installation by setting the `restdefinitions` field in the `values.yaml` file. 
This may be useful if you want to limit the resources managed by the provider to only those you need, reducing the overhead of managing unnecessary controllers.
The default configuration enables all resources supported by the chart.

### Verbose logging

In order to enable verbose logging for the controllers, you can add the `krateo.io/connector-verbose: "true"` annotation to the metadata of the resources you want to manage, as shown in the examples above. 
This will enable verbose logging for those specific resources, which can be useful for debugging and troubleshooting as it will provide more detailed information about the operations performed by the controllers.

## Chart structure

Main components of the chart:

- **RestDefinitions**: These are the core resources needed to manage resources leveraging the Krateo OASGen Provider. In this case, they refers to the OpenAPI Specification to be used for the creation of the Custom Resources (CRs) that represent Azure DevOps resources.
They also define the operations that can be performed on those resources. Once the chart is installed, RestDefinitions will be created and as a result, specific controllers will be deployed in the cluster to manage the resources defined with those RestDefinitions.

- **ConfigMaps**: Refer directly to the OpenAPI Specification content in the `/assets` folder.

- **/assets** folder: Contains the selected OpenAPI Specification files for the Azure DevOps REST API.

- **/samples** folder: Contains example resources for each supported resource type as seen in this README. These examples demonstrate how to create and manage Azure DevOps resources using the Krateo Azure DevOps Provider KOG.

- **Deployment**: Deploys a [plugin](https://github.com/krateoplatformops/azuredevops-rest-dynamic-controller-plugin) that is used as a proxy to resolve some inconsistencies of the Azure DevOps REST API. The specific endpoins managed by the plugin are described in the [plugin README](https://github.com/krateoplatformops/azuredevops-rest-dynamic-controller-plugin/blob/main/README.md)

- **Service**: Exposes the plugin described above, allowing the resource controllers to communicate with the Azure DevOps REST API through the plugin, only if needed.

## Troubleshooting

For troubleshooting, you can refer to the [Troubleshooting guide](./docs/troubleshooting.md) in the `/docs` folder of this chart. 
It contains common issues and solutions related to this chart.
