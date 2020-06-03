---
title: "Unidex Demo"
description: "Introduce a sample project to explore devops with .Net Core and public CSPs."
date: 2020-05-23T19:54:31-05:00
sidebar: right
draft: true
authorbox: true
tags:
  - azure
  - terraform
  - .net-core
  - architecture
---
This project is simply exploratory around using various DevOps tools within the .Net Core ecosystem
and fulfilling some enterprise like functionality. Unidex is a toy application for building a
fictional universe, but not really a focus here. Its functionality will be minimal.
[Terraform](https://www.terraform.io/docs/index.html) will be used for provisioning and
configuration. [Github Actions](https://github.com/features/actions) will be leveraged for CI/CD. Initially
cloud infrastructure will be on Azure.

## Infrastructure as Code

Terraform has extensive capabilities in provisioning infrastructure. But it can be used
additionally for provisioning services and configurations. The intent here is to maximize its
use to gain declarative, predictable, and automated deployments.

  * Deploy identity information to Azure AD
  * Deploy application components on Azure PaaS offerings
  * Configure Github Action secrets to be consistent with Azure AD identities
  * Create Cloudflare CDN proxies
  * Integrate one or more OIDC identity providers
  * Additional components as they come to mind

## Platform-as-a-Service

All major public cloud service providers (CSPs) offer not only API-driven deployment of virtual
infrastructure, but managed layers on top of that infrastructure. Managed application runtimes, 
database instances, and API management proxies. While not always a great fit, they tend to offer
many times the efficiency gain over simply moving where you spin-up VMs.

## Strong Separation of Concerns

It is incredibly easy to fall into vendor lock-in with cloud services. As a provider's service
offerings expand, almost always it is easier to couple those services with anything already in use.
Rather than falling into this pattern, we'd rather have clear boundaries between functional domains and
require standard interfaces between domains. This is why a tool like Terraform is critical as it
allows use of many providers in a single architecture and maintains consistency.
