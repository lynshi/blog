---
title: Purging Azure CDN With GitHub Actions
description: >-
  GitHub Actions allows you to automate purging Azure CDN so you don’t have to
  craft caching rules or manually enter the Azure portal.
date: '2020-01-01T04:13:58.567Z'
slug: /@shilyndon/purging-azure-cdn-with-github-actions-1c18e2adaf18
categories:
  - Deployment
tags:
  - Microsoft Azure
  - GitHub Actions
  - Automation
  - Website
---

When you use Azure CDN to deliver content, you may need to purge your endpoint so that changes you make are sent to your users, as [files are cached in the Azure CDN until their time-to-live (TTL) expires](https://docs.microsoft.com/en-us/azure/cdn/cdn-manage-expiration-of-cloud-service-content). If you don’t set a TTL for your files, Azure automatically sets a TTL of 7 days. Even if you set a lower TTL, your updates may not coincide with the cache expiration.

For example, my personal website uses Azure CDN and is updated on every push I make to GitHub. It’s tough to set a good caching rule for this because my changes are unpredictable. When I’m working on my site, I might push several times a day, but I can also go weeks without modifying my website.

While it is possible to [purge an endpoint through the Azure portal](https://docs.microsoft.com/en-us/azure/cdn/cdn-purge-endpoint), I wanted to automate this process. Fortunately, this is possible with [GitHub Actions](https://github.com/features/actions).

#### Creating a Service Principal

An [Azure service principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?toc=%2Fazure%2Fazure-resource-manager%2Ftoc.json&view=azure-cli-latest) is “an identity created for use with applications, hosted services, and automated tools to access Azure resources”. It is recommended, for better security, to use service principals with automated tools, since access is restricted by roles assigned to the service principal. In this section, I’ll discuss how to create a service principal and give it the [CDN Endpoint Contributor](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#cdn-endpoint-contributor) role. To get started, you’ll need the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest).

To create a new service principal, enter the following command in your terminal, making sure to copy the correct`<subscription_id>` and `<resource_group_name>` from your Azure portal.

```bash
az ad sp create-for-rbac -n "$nameOfServicePrincipal" --role "CDN Endpoint Contributor" --sdk-auth --scopes "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup"
```

The output will look similar to the following; be sure to copy and save it somewhere safe, as you will not be able to retrieve the `clientSecret` in the future.

```json
{
  "clientId": "<GUID>",                               
  "clientSecret": "<GUID>",                           
  "subscriptionId": "<GUID>",                         
  "tenantId": "<GUID>"  
}
```

In the [command](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create), we are creating a service principal and assigning it a role of “CDN Endpoint Contributor”. This allows the service principal to manage CDN endpoints. The `--sdk-auth` flag causes the output to be correctly formatted for use in the GitHub Action; without it, you will get the following error when you copy the output to GitHub.

```
Error: Not all values are present in the creds object. Ensure clientId, clientSecret, tenantId and subscriptionId are supplied.
```

#### Adding GitHub Secrets

Add the following secrets (encrypted environment variables) to your GitHub repository:

1.  `AZURE_CREDENTIALS`: paste the output from the `az ad sp create-for-rbac` command.
2.  `AZURE_CDN_ENDPOINT`: the name of your Azure CDN endpoint.
3.  `AZURE_CDN_PROFILE_NAME`: the name of your Azure CDN profile.
4.  `AZURE_RESOURCE_GROUP`: the name of the resource group containing the Azure CDN profile.

![](/img/medium/1__RwM3IOTXVjwtQCcMkDWW__A.png)

#### Writing the GitHub Actions Workflow File

Now that you have a service principal and GitHub secrets, all that remains is to write the GitHub Action. There are two essential steps in the workflow: 1) logging in to Azure, and 2) purging the CDN. To start, go to your repository on GitHub and select the “Actions” tab. Click “New workflow”, then choose “Set up a workflow yourself” or the appropriate starter for your needs.

The following is the bare minimum required for this workflow.

The `Azure service principal login` step logs in to Azure CLI with the credentials you added to your GitHub secret. The `Purge CDN` step purges the CDN, so your updates should occur between these two steps. In the `Purge CDN` step, `--content-paths "/*"` purges all content. `--no-wait` means the script does not wait for the purge operation to finish before continuing; this flag is optional. You can read more about the `az cdn endpoint purge` command [here](https://docs.microsoft.com/en-us/cli/azure/cdn/endpoint?view=azure-cli-latest#az-cdn-endpoint-purge).

That’s all there is to it, happy coding!