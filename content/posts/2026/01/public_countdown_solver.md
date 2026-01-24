+++
date = '2026-01-24T09:27:32Z'
draft = true
title = 'Public Countdown Solver'
tags = ['programming', 'rust', 'wasm', 'artificial intelligence']
ShowToc = true
TocOpen = true
+++

## Introduction

In this third part of our ongoing "Countdown Numbers Round Solver" series, I'll be making the solver available for you (and others) on the Internet. This means that you'll be able to visit the [Countdown Solver](https://cds.newman.digital) and challenge yourself with random problems, or request a solution to a particular problem. Ideally I'd like to make this cost as little as possible for me, so I want to serve the solver using [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static), which means we'll need to make sure the computation involved when solving the problem happens on the user's device, not a compute element in Azure (which I'd be paying for). There are a number of options we could leverage, from any number of JavaScript UI frameworks/libraries (such as [Angular](https://angular.dev/), [React](https://react.dev/), or [Vue](https://vuejs.org/guide/introduction)) or we could use [ASP.NET Core Blazor's](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor) [WebAssembly hosting model](https://learn.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-10.0#blazor-webassembly) as we already have a functioning C# implementation. Leveraging Blazor would certainly be the quickest "route to market", but let's push ourselves to do something a little more outside of our comfort zone. You can navigate to the code associated with this post at [yew-countdown-solver](https://github.com/newmancodes/yew-countdown-solver).

### AI Declaration

I have made use of multiple Generative AI solutions during this project. During the writing process I have consulted with Anthropic's Claude Sonnet 4.5 model via GitHub Copilot (in Visual Studio Code) to help me construct a good flow to the article and perform a content review. During the coding portion of the project I made use of both GitHub Copilot and Claude Code, I typically use Anthropic's Claude Sonnet 4.5 for most of my Gen-AI interactions. I use Claude Code to help me navigate issues with Rust, Yew, and Trunk. Occasionally generating plans and executing those plans (you can find these in the GitHub repo at [./.claude/plans](https://github.com/newmancodes/yew-countdown-solver/blob/main/.claude/plans)). Beyond these usages I have crafted this project by hand and enjoyed the ride. When these technologies helped me over a particular hurdle, I'll call it out below rather than leaving a bland "I prompted" and leaving it as an exercise to the reader to discover how to do that for themselves. There are a lot of people out there that are quick to comment "skill issue! you're using it wrong", but we're all learning these tools in parallel and I certainly haven't gotten this all figured out myself, let's share what's worked for us and we'll all improve together and figure out where the sharp edges are.

## WebAssembly

> WebAssembly (abbreviated Wasm) is a binary instruction format for a stack-based virtual machine. Wasm is designed as a portable compilation target for programming languages, enabling deployment on the web for client and server applications.
>
> -- <cite>[WebAssembly.org](https://webassembly.org/)</cite>

Now, I'm not going to be programming this code in the raw binary instruction format that WebAssembly executes! This means I need to find a programming language which I can leverage and produce a WASM build artifact that can run on the virtual machine (VM). I've already mentioned C# as a possible option (via Blazor), but there are several options I could choose. I could use C, C++, Go, Python, Java, and many others, but I'm going to be using Rust for this project.

## Rust

Why Rust?

What even is Rust?

The Rust compiler can compile straight to WebAssembly when the wasm32_unknown_unknown target is added by executing the command `rustup target add wasm32-unknown-unknown`.

## Yew + Trunk

### Tailwind CSS

## Implementing in Rust

## Playwright

## Deploying to Azure

My Azure setup is fairly simple, I maintain a single Subscription and I tend to deploy various elements into different resource groups for logical separation. I'll be using [Azure Deployment Stacks](https://learn.microsoft.com/en-us/training/modules/introduction-to-deployment-stacks/) for this project so that the only mechanism to make changes to my cloud infrastructure is via a deployment (using [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep)), this prevents [configuration drift](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/management-operational-compliance#monitor-for-configuration-drift), and also so I can easily tear down the infrastructure via an `az stack` command. If you want to learn more about Azure Deployment Stacks, John Savill posted a great [Deployment Stacks Deep Dive](https://youtu.be/d1AE8qLwBYw?si=2LJpwiXTFcFAqUv9) video to YouTube.

When I configure my GitHub Actions based deployments to Azure I make use of the federation setup between GitHub and Azure, this means I don't need to worry about handling secrets which may expire. This allows my deployments to reliably work, whenever I want to trigger them. I start the process by creating a new App Registration using the Azure CLI.

```bash
appId=$(az ad app create --display-name countdown-solver-deployer --sign-in-audience AzureADMyOrg --query appId -o tsv)
```

This command creates a new App Registration within my tenant and populates the id of the new application into appId variable so I can use it in the following steps in the process. I then need to configure the federated credentials for my GitHub repo. This is a two step process, first I create a [json document](https://github.com/newmancodes/yew-countdown-solver/blob/main/deploy/credential.json) which describes the particulars about the federated credential. The most important part in this document is the `subject` value:

```json
{
    // redacted
    "subject": "repo:newmancodes/yew-countdown-solver:ref:refs/heads/main",
    // redacted
}
```

This subject will allow this project's repository to authenticate with Azure as long as the context is associated with the main (default) branch. I don't envisage requiring multiple environments for this project, nor will I make use of extra branches so only allowing deployments of code that has made it into main makes sense to me. In order to associate this federated credential with the newly created App Registration, I'll execute this az command from the directory containing my `credential.json` file.

```bash
az ad app federated-credential create --id $appId --parameters credential.json
```

Note: This can take some time to appear in the Azure Portal so don't worry if you're following along and it hasn't appeared immediately. While I wait can execute the following statements in order:

- Create a Service Principal associated with my new App Registration, this is so I have a target to make Role-Based Access Control (RBAC) Role Assignments.
  ```bash
  az ad sp create --id $appId
  ```
- Create the Resource Group to contain the logical set of infrastructure necessary for this project.
  ```bash
  az group create --name rg-countdown-solver --location uksouth
  ```
- Perform the RBAC Role Assignment so that when the GitHub Actions workflow uses the `azure/login` action, it will be able to manage the Azure Deployment Stack
  ```bash
  subscriptionId=$(az account show --query id -o tsv) && az role assignment create --assignee $appId --role "Azure Deployment Stack Owner" --scope "/subscriptions/$subscriptionId/resourceGroups/rg-countdown-solver"
  ```
- Store the secrets used by the GitHub Actions workflow when authenticating with Azure. Note: I am using the [GitHub CLI](https://cli.github.com/) here.
  ```bash
  gh secret set AZURE_DEPLOYMENT_APP_CLIENT_ID --body "$appId"
  ```
  ```bash
  gh secret set AZURE_DEPLOYMENT_APP_SUBSCRIPTION_ID --body "$subscriptionId"
  ```
  ```bash
  tenantId=$(az account show --query tenantId -o tsv) && gh secret set AZURE_DEPLOYMENT_APP_TENANT_ID --body "$tenantId"
  ```



### Security

## Conclusion
