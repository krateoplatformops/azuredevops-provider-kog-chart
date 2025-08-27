# Krateo Azure DevOps Provider KOG - Troubleshooting Guide

This document serves as a troubleshooting guide for the Krateo Azure DevOps Provider KOG.

## Summary

- [Summary](#summary)
- [GitRepository](#gitrepository)
  - [1. New Repository Cases](#1-new-repository-cases)
    - [Case 1.1: New Repo Without Initialization](#case-11-new-repo-without-initialization)
    - [Case 1.2: New Repo With Initialization (Default Branch Omitted)](#case-12-new-repo-with-initialization-default-branch-omitted)
    - [Case 1.3: New Repo With Initialization and Custom Default Branch](#case-13-new-repo-with-initialization-and-custom-default-branch)
    - [Case 1.4: New Repo With Custom Default Branch (Without Initialization Flag set to `true`)](#case-14-new-repo-with-custom-default-branch-without-initialization-flag-set-to-true)
  - [2. Forked Repository Cases (Parent Repository Initialized)](#2-forked-repository-cases-parent-repository-initialized)
    - [Case 2.1: Forked Repo With Nonexistent SourceRef](#case-21-forked-repo-with-nonexistent-sourceref)
    - [Case 2.2: Forked Repo With Existing Default Branch in Parent](#case-22-forked-repo-with-existing-default-branch-in-parent)
    - [Case 2.3: Forked Repo With Nonexistent Default Branch (Fallback to Parent Default)](#case-23-forked-repo-with-nonexistent-default-branch-fallback-to-parent-default)
  - [3. Forked Repository Cases (Parent Repository Not Initialized)](#3-forked-repository-cases-parent-repository-not-initialized)
    - [Case 3.1: Fork With Invalid `sourceRef`](#case-31-fork-with-invalid-sourceref)
    - [Case 3.2: Fork With No Branch Configuration](#case-32-fork-with-no-branch-configuration)
    - [Case 3.3: Fork With Nonexistent Default Branch](#case-33-fork-with-nonexistent-default-branch)

## GitRepository 

This section outlines the expected behaviors of GitRepositories under different configurations during creation and forking.
Each case states the input configuration and resulting behavior.

Some sample Custom Resources are provided in the [`/samples/gitrepository/` directory](../chart/samples/gitrepository/) of the chart. 
Files are named after some of the cases described below, e.g., [`gitrepository_2.3.yaml`](../chart/samples/gitrepository/gitrepository_2.3.yaml) corresponds to Case 2.3.

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

#### Case 1.3: New Repo With Initialization and Custom Default Branch
- **Input**:
  - `initialize`: `true`
  - `defaultBranch`: `test-branch`
- **Result**:
  - Repository is initialized with a **first commit** on the `test-branch` branch.

---

#### Case 1.4: New Repo With Custom Default Branch (Without Initialization Flag set to `true`)
- **Input**:
  - `initialize`: `false` or *omitted*
  - `defaultBranch`: `test-branch`
- **Result**:
  - Error 400: `When specifying a 'defaultBranch' for a new repository, 'initialize' must be set to true`

---

### 2. Forked Repository Cases (Parent Repository Initialized)

> In fork scenarios, the `initialize` field is **ignored**.

#### Case 2.1: Forked Repo With Nonexistent SourceRef
- **Preconditions**:
  - Parent repository is **initialized** but does **not** have a branch named `test-branch`.
- **Input**:
  - `defaultBranch`: anything or omitted
  - `sourceRef`: `refs/heads/test-branch-not-existing`
- **Result**:
  - **Error 400**: `SourceRef 'refs/heads/test-branch-not-existing'` does not exist in parent repository.

---

#### Case 2.2: Forked Repo With Existing Default Branch in Parent
- **Preconditions**:
  - Parent repository is **initialized** and has a branch named `test-branch` (either as default or non-default).
- **Input**:
  - `defaultBranch`: `test-branch` (which exists in parent repo)
  - `sourceRef`: *includes* `test-branch` or *omitted*
- **Result**:
  - Repository is forked with `test-branch` as the **default branch** (git history is copied). Note that in the parent repo, `test-branch` can be the default branch or any other branch.

---

#### Case 2.3: Forked Repo With Nonexistent Default Branch set (Fallback to Parent Default)
- **Preconditions**:
  - Parent repository is **initialized** and has a default branch (e.g., `main`) but does **not** have a branch named `test-branch`.
- **Input**:
  - `defaultBranch`: `test-branch` (does **not** exist in parent)
  - `sourceRef`: *omitted* or references an **existing** branch in the parent
- **Result**:
  - Repository is forked with the **default branch of the parent** repository (e.g., `main`) as the default branch (git history is copied).
  - Status code: **202 Accepted**
  - Notes: In the context of the `gitrepository-controller`, the controller awaits the manual creation of the `test-branch` by the user. 
  The Kubernetes resource will be in a `Ready: False` state until the user creates the `test-branch` on Azure DevOps.
  After the user manually creates the `test-branch` on the GitRepository (not in the parent), the `gitrepository-controller` will update the repository to set `test-branch` as the default branch. Finally, the GitRepository CR will become `Ready: True`.

---

### 3. Forked Repository Cases (Parent Repository Not Initialized)

Note: it is not recommended to fork from uninitialized repositories.

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
  - Notes: In the context of the `gitrepository-controller`, the controller awaits the manual creation of the `test-branch` by the user. 
  The Kubernetes resource will be in a `Ready: False` state until the user creates the `test-branch` on Azure DevOps.
  After the user manually creates the `test-branch` on the GitRepository (not in the parent), the `gitrepository-controller` will update the repository to set `test-branch` as the default branch. Finally, the GitRepository CR will become `Ready: True`.

---
