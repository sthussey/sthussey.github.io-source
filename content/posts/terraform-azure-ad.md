---
title: "Terraform and Azure Identities"
description: "Build the base of an applicaiton deployment by configuring Azure AD and Managed Identities via Terraform."
date: 2020-05-13T19:57:07-05:00
draft: true
sidebar: right
authorbox: true
tags:
  - azure
  - terraform
---
[Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/) (AAD) is Microsoft's cloud-based identity and
access management (IAM) solution. It can be used as a standalone directory service and identity provider or
as an extension of an on-premise Microsoft AD Domain Controller. This post isn't meant to delve to deeply
into all that AAD can do. Rather, how one can use [Terraform](https://www.terraform.io/docs/index.html)
to manage the contents of a small AAD instance to manage access to deployed web applications.

## Identity Design

![Identity Architecture](/img/Unidex-Overview.png)

There are three types of identities used here, all of them managed by an Azure AD instance. Terraform can manage
them all using the [Azure Active Directory provider](https://www.terraform.io/docs/providers/azuread/index.html), however
one identity with administrative privileges will need to be pre-provisioned for Terraform to use when accessing the Azure
API.

  * Service Principal - A service principal is an Azure AD identity in a single tenant that is intended to represent an
    application. Microsoft [documents the comparison of an application and a service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals),
    but a quick metaphor is that an application is like a job description shared among many businesses (tenants).
    A service principal is like an employee within each business fulfilling that job description.
  * Managed Identity - This is an automation layer on top of a service principal. Rather than requiring the user to
    create a service principal for a deployed cloud resource, Azure Resource Manager will create a managed identity
    meaning that it does the work to create the service principal in Azure AD, assign the service principal to the
    resource, and assign the needed permissions. When a managed idenity is created, the associated service principal
    is still viewable in Azure AD.
  * User Principal - This identity represents a real person. Initially this is the only principal in Azure AD when a
    new tenant is created.

In our identity design, we have one user principal (the user implementing the project), one direct service principal
(the identity that deploys code), and one managed identity (the identity the web app runs as). As more resource types
are added to the architecture, additional identities with proper cross-resource permissions will be added.

## Prepare 

Some configuration needs to be in place prior to starting. First we assume that you already have accounts created in
the services we'll be using.

  * Azure - An [Azure free account](https://azure.microsoft.com/en-us/free/) is needed to provision all the Azure resources.
    All of the resources fit in the Azure always free tier (assuming you don't already have anything else created), so this
    project does not cost anything.
  * Github - Both a personal account as well as a [Github organization](https://help.github.com/en/enterprise/2.20/admin/user-management/creating-organizations) are
    needed. The organization is only needed due to a [limitation](https://github.com/terraform-providers/terraform-provider-github/issues/422) of the Terraform Github provider.
    All code will be pushed to a repository under the organizaiton, not a personal account. And the repo will need to have
    Github Actions enabled. Lastly, you'll need a [Github personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) 
    for Terraform to use as a credential.

Locally, you'll need to have git, [Terraform](https://learn.hashicorp.com/terraform/getting-started/install), and the Azure [az CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed.
Though technically not required, if you want to actually run your .Net application code locally you'll need the [.Net Core 3.1 SDK](https://dotnet.microsoft.com/download/dotnet-core/3.1).

The Terraform Azure providers require you have already authenticated your az CLI with credentials with administrative privileges in the
target tenant. You can use a personal account or a service principal - any samples here will be using a personal account.

## 1 - Provision Resources

[Github Code Reference](https://github.com/sthussey-net/unidex-demo/tree/master/deploy)

### Bootstrapping Terraform

Terraform will be used to create all resources in use by our web app. We'll use [HCL](https://www.terraform.io/docs/configuration/syntax.html) to describe
our resources.

## 2 - .Net Core Code

## 3 - Test and Build

## 4 - Deploy

