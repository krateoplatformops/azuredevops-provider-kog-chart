# AzureDevOps Provider KOG - Troubleshooting Guide

## GitRepository 

This section outlines the expected behaviors of GitRepositories under different configurations during creation and forking.
Each case states the input configuration and resulting behavior.

Sample Custom Resources are provided in the [`/samples/gitrepository/` directory](../chart/samples/gitrepository/) of the chart. 
Files are named after the cases described below, e.g., [`gitrepository_2.3.yaml`](../chart/samples/gitrepository/gitrepository_2.3.yaml) corresponds to Case 2.3.

---

### 1. New Repository Cases

#### Case 1.1: New Repo Without Initialization
- **Input**:
  - `initialize`: *omitted*
  - `defaultBranch`: *omitted*
- **Result**: 
  - Repository is created **empty**, with **no branches**.

---

#### Case 1.2: New Repo With Initialization (Default Branch Omitted)
- **Input**:
  - `initialize`: `true`
  - `defaultBranch`: *omitted*
- **Result**:
  - Repository is initialized with a **first commit** on the `main` branch (which is set as `defaultBranch` as it is the default branch in Azure DevOps).

---

#### Case 1.3: New Repo With Custom Default Branch (Without Initialization Flag set to `true`)
- **Input**:
  - `initialize`: `false` or *omitted*
  - `defaultBranch`: `test-branch`
- **Result**:
  - Repository is **auto**-initialized with a **first commit** on the `test-branch` branch (which is set as `defaultBranch`).
  - Notes: in this case the plugin initializes the repository in order to set the `test-branch` as the default branch with a first commit. So setting `initialize` to `false` and `defaultBranch` to any branch is not a recommended practice as the values are logically contradictory.
---

### 2. Forked Repository Cases (Parent Repository Initialized)

> In fork scenarios, the `initialize` field is **ignored**.

#### Case 2.1: Forked Repo With Existing Target Branch
- **Preconditions**:
  - Parent repository is **initialized** and has a branch named `test-branch`.
- **Input**:
  - `defaultBranch`: `test-branch` (which exists in parent repo)
  - `sourceRef`: *includes* `test-branch` or *omitted*
- **Result**:
  - Repository is forked with `test-branch` as the **default branch** (git history is copied).

---

#### Case 2.2: Forked Repo With Nonexistent SourceRef
- **Preconditions**:
  - Parent repository is **initialized** but does **not** have a branch named `test-branch`.
- **Input**:
  - `defaultBranch`: anything or omitted
  - `sourceRef`: `test-branch`
- **Result**:
  - **Error 400**: `SourceRef 'refs/heads/test-branch'` does not exist in parent repository.

---

#### Case 2.3: Forked Repo With Nonexistent Default Branch (Fallback to Parent Default)
- **Preconditions**:
  - Parent repository is **initialized** and has a default branch (e.g., `main`) but does **not** have a branch named `test-branch`.
- **Input**:
  - `defaultBranch`: `test-branch` (does **not** exist in parent)
  - `sourceRef`: *omitted* or references an **existing** branch in the parent
- **Result**:
  - Repository is forked with the **default branch of the parent** repository (e.g., `main`) as the default branch (git history is copied).
  - Status code: **202 Accepted**
  - Notes: In the context of the `gitrepository-controller`, the controller awaits creation of the `test-branch` by the user. After the user creates the `test-branch`, the `gitrepository-controller` will update the repository to set `test-branch` as the default branch. Finally, the GitRepository CR will become `Ready: True`.

---

### 3. Forked Repository Cases (Parent Repository Not Initialized)

> In fork scenarios, the `initialize` field is **ignored**.

#### Case 3.1: Fork With Invalid `sourceRef`
- **Preconditions**:
  - Parent repository is **not initialized** (no branches exist).
- **Input**:
  - Parent repo is **not initialized**
  - `defaultBranch`: *omitted*
  - `sourceRef`: `test-branch-not-existing`
- **Result**:
  - **Error 400**: `SourceRef 'refs/heads/test-branch-not-existing'` does not exist in parent repository.

---

#### Case 3.2: Fork With No Branch Configuration
- **Input**:
  - Parent repo is **not initialized**
  - `defaultBranch`: *omitted*
  - `sourceRef`: *omitted*
- **Result**:
  - Repository is forked in an **uninitialized** state.
  - No branches are created.

---

#### Case 3.3: Fork With Nonexistent Default Branch
- **Input**:
  - Parent repo is **not initialized**
  - `defaultBranch`: `test-branch`
  - `sourceRef`: *omitted*
- **Result**:
  - Repository is forked in an **uninitialized** state.
  - No branches are created.
  - Status code: **202 Accepted**
- Notes: In the context of the `gitrepository-controller`, the controller awaits creation of the `test-branch` by the user. After the user creates the `test-branch`, the `gitrepository-controller` will update the repository to set `test-branch` as the default branch. Finally, the GitRepository CR will become `Ready: True`.

---
