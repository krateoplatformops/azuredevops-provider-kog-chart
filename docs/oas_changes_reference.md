# Krateo Azure DevOps Provider KOG - OpenAPI Specification (OAS) Changes reference

This documents serves as a reference for the changes made to the OpenAPI Specification (OAS) of the resources managed by the Azure DevOps Provider KOG.
Note that the changes are made to comply with some requirements of the Krateo Rest Dynamic Controller or to fix issues with the Azure DevOps REST API.

## Summary

- [Summary](#summary)
- [GitRepository](#gitrepository)
- [Pipeline](#pipeline)
- [PipelinePermission](#pipelinepermission)

## `GitRepository`

Version: 7.2-preview.2

Specification file:
https://github.com/MicrosoftDocs/vsts-rest-api-specs/blob/ed36f85e1796b78f1c88961a45396e31c6618000/specification/git/7.2/git.json

The path parameter `project` has been changed to `projectId` on every endpoint that requires it.
Otherwise there would be a potential clash between `project` field in path and `project` field in the response body that would cause issues with Rest Dynamic Controller.
Note that `projectId` could be either a project name or a project ID even if the field is named `projectId`.

The path parameter `repositoryId` has been changed to `id` on every endpoint that requires it. This is done to align with the naming convention used in response bodies.

In the `delete` endpoint the response status code has been changed from `200` to `204` as the Azure DevOps REST API actually returns a `204 No Content` status code when a repository is deleted successfully.

Some schemas were created, such as `GitRepositoryUpdateOptions`. 
This schema is used in the `update` operation of the `GitRepository` resource to allow updating only the name and default branch of a Git repository.

## `Pipeline`

Version: 7.2-preview.1

The schema for the request body of the `create` operation has been modified to include additional fields not documented in the original OpenAPI Specification (OAS) but required for a successful operation (`configuration.repository` and `configuration.path`).


Note: Build Definitions are used in order to perform `update` and `delete` operations which are not available for Pipelines.Build Definitions use version 7.2-preview.7

## `PipelinePermission`

Version: 7.2-preview.1

Since the Azure DevOps REST API returns only the pipelines that are authorized for the user, the `PipelinePermission` resource of this provider allows you to set the `authorized` field of each `pipeline` in the `pipelines` array to `true` only.
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

An `enum` has been added to the `resourceType` field for both the GET and PATCH operations:
```diff
  - name: resourceType
    in: path
    required: true
    schema:
      type: string
+     enum:
+       - repository
+       - environment
+       - queue
+       - teamproject
+       - endpoint
+       - variablegroup
+       - securefile
```