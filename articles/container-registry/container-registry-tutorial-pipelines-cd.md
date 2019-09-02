---
title: Continuous deployment in Azure Pipelines
description: Learn how to setup continuous deployment triggers in Azure Pipelines for push image events to Azure Container Registry
services: container-registry
author: shashankbarsin
manager: gwallace

ms.service: container-registry
ms.topic: quickstart
ms.date: 08/17/2019
ms.author: shasb
---

# Continuous deployment in Azure Pipelines

In this step-by-step guide you'll learn how to setup continuous deployment in Azure Pipelines that can be triggered by push of images to Azure Container Registry. This guide uses an Azure Kubernetes Service cluster as the choice of deployment target. However, with release pipelines in Azure DevOps just being a means to run an automated sequence of steps, it is possible to use these pipelines to deploy to Web App for Container, Azure Container Instance, or even perform custom actions using command line tasks.

Triggering release pipelines in Azure DevOps involves addition of an Azure Container Registry artifact along with enabling of the continuous deployment trigger on this artifact. The release pipeline is able to watch push events by registering a webhook on the chosen repository within the container registry. The following section utilizes the build pipeline setup as a part of [Continuous Integration in Azure Pipelines](./container-registry-tutorial-pipelines-build.md) to push images, which in turn triggers off the release pipeline. However, note that it is also possible to trigger release pipelines using equivalent push image action from any other CI pipeline or even from CLI.

## Update manifest files
For the [MicrosoftDocs/azure-pipelines-canary-k8s](https://github.com/MicrosoftDocs/azure-pipelines-canary-k8s) repository forked as a part of the CI guide, replace `<foobar>` with the name of your container registry in `manifests/deployment.yml` file. Push the commits to your forked repository.

## Create service connections
1. Navigate to Project settings -> Pipelines -> Service connections.
2. Create a [Kubernetes service connection](https://docs.microsoft.com/azure/devops/pipelines/library/service-endpoints?view=azure-devops#sep-kuber) for the Kubernetes cluster and namespace you want to deploy to. Name it **k8sServiceConnection**.

## Add artifacts
1. Navigate to **Pipelines** -> **Releases**, click on **+ New** -> **New release pipeline**.
2. Click on **+ Add** in the artifacts pane. Choose **Azure Container Registry** as source type.
3. For **Service connection** field, choose the Azure Resource Manager service connection corresponding to your subscription. If you don't already have a service connection, click on **Manage** to create one.
4. Choose the appropriate resource group, registry and repository. Use **Latest** as the default version.
5. For **Source alias** field, use the value **image**. Click on **Add** to complete addition of this artifact.
6. Once the artifact has been added, click on the lightning bolt icon on the artifact card to enable continuous deployment trigger. 
7. Click on **+ Add** in the artifacts pane. Choose **GitHub** as source type.
8. For **Service connection** field, choose a GitHub service connection. If you don't already have a service connection, click on **Manage** to create one.
9. Select the repository, branch and use **Latest from the default branch** for **Default version** field.
10. For **Source alias** field, use the value **azure-pipelines-k8s**. Click on **Add** to complete addition of this artifact.

## Add deploy stage
1. Add a stage to the pipeline, and choose the **Empty job** template when presented with the options. Name the stage as **Deploy**
2. In **Deploy** stage that you just created, click on **1 job, 0 task** link to be navigated to the window for adding jobs and tasks.
3. Click on **Agent job**. In the configuration window, in the **Agent pool** dropdown window, choose **Hosted Ubuntu 1604**.
4. Click on the '+' on the agent job row to add a new task. Add **Deploy Kubernetes manifests** task with the following configuration -
    - **Display name**: Create secret
    - **Action**: create secret
    - **Kubernetes service connection**: k8sServiceConnection
    - **Namespace**: namespace within the cluster you want to deploy to
    - **Type of secret**: dockerRegistry
    - **Secret name**: azure-pipelines-k8s
    - **Docker registry service connection**: acrServiceConnection

5. Add another **Deploy Kubernetes manifests** task with the following configuration -
    - **Display name**: Deploy
    - **Action**: deploy
    - **Kubernetes service connection**: k8sServiceConnection
    - **Namespace**: namespace within the cluster you want to deploy to
    - **Strategy**: None
    - **Manifests**: azure-pipelines-k8s/manifests/*
    - **Containers**: foobar.azurecr.io/azure-pipelines-k8s:$(Release.Artifacts.image.BuildId) where `foobar` is to be replaced with the name of the Azure Container Registry.
    - **ImagePullSecrets**: azure-pipelines-k8s
6. Click on **Save** to save the release pipeline definition.

## See it in action
In `app/app.py`, change the `success_rate` variable to a different value. Follow it up by building and pushing the image to Azure Container Registry. This will result in a new release being created for the release pipeline you had created earlier!