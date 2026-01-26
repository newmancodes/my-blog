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

When I configure my GitHub Actions based deployments to Azure I make use of the federation setup between GitHub and Azure. This means I don't need to worry about handling secrets which may expire. This allows my deployments to reliably work, whenever I want to trigger them. I need to create an App Registration that supports GitHub federated authentication, for that I am going to need to express the federated credential as a [json document](https://github.com/newmancodes/yew-countdown-solver/blob/main/deploy/credential.json) which I will supply in one of the upcoming commands. Here is an example of the credential.json file:

```json
{
  "name": "GitHub",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:newmancodes/yew-countdown-solver:ref:refs/heads/main",
  "description": "Manages deployments of the countdown solver project from GitHub",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
```

The subject property here is pretty confusing, so warrants some explanation. This property allows me to instruct Azure to check the subject claim presented by GitHub to make sure that:

- The correct repository owner is described `newmancodes`
- The correct repository is described `yew-countdown-solver`
- The correct branch is described `main`

By configuring this subject property I can control precisely which kind of CI/CD processes can authenticate with Azure. This works because of the federated trust relationship between Azure and GitHub. If in your projects you need to apply different rules I would recommend first using the Azure Portal and experimenting with various options (especially in regards to Scope). For this project I don't feel the need to support different branches or environments so I can apply a simple "it must come from main" declaration. Now that I have the credential.json file specified I can begin to use the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) and [GitHub CLI](https://cli.github.com/) to execute the commands required to:

- Support Azure authentication from my GitHub Actions workflow
- Create the containing Resource Group and apply both Contributor and Azure Deployment Stack Owner Role-Based Access Control (RBAC) Role Assignments
- Store the required GitHub repository secrets so the [azure/login](https://github.com/marketplace/actions/azure-login) GitHub Action can attempt to authenticate

```bash
# Create a new app registration and extract the Application (Client Id)
# of the application.
appId=$(az ad app create \
    --display-name countdown-solver-deployer \
    --sign-in-audience AzureADMyOrg \
    --query appId -o tsv)

# Add the GitHub Action federated credential to the newly created
# App Registration.
az ad app federated-credential create \
    --id $appId \
    --parameters credential.json

# Create a Service Principal associated with the new App Registration
# to support RBAC Role Assignments.
az ad sp create --id $appId

# Create the Resource Group, I've selected westeurope as
# Static Web Apps aren't available everywhere.
az group create \
    --name rg-countdown-solver \
    --location westeurope

# Create an RBAC Role Assignment that grants the Service Principal
# both the Contribtutor and Azure Deployment Stack Owner roles
# scoped to our newly created Resource Group.
subscriptionId=$(az account show --query id -o tsv)
az role assignment create \
    --assignee $appId \
    --role "Contributor" \
    --scope "/subscriptions/$subscriptionId/resourceGroups/rg-countdown-solver"
az role assignment create \
    --assignee $appId \
    --role "Azure Deployment Stack Owner" \
    --scope "/subscriptions/$subscriptionId/resourceGroups/rg-countdown-solver"

# Store repository level secrets using the GitHub CLI to be used
# when authenticating with Azure.
tenantId=$(az account show --query tenantId -o tsv) 
gh secret set AZURE_DEPLOYMENT_APP_CLIENT_ID --body "$appId"
gh secret set AZURE_DEPLOYMENT_APP_SUBSCRIPTION_ID --body "$subscriptionId"
gh secret set AZURE_DEPLOYMENT_APP_TENANT_ID --body "$tenantId"
```

### Security

## Conclusion
