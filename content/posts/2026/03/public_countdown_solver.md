+++
date = '2026-03-17T10:00:00Z'
draft = true
title = 'Public Countdown Solver'
tags = ['programming', 'rust', 'wasm', 'python', 'azure', 'github actions', 'artificial intelligence']
ShowToc = true
TocOpen = true
+++

## Introduction

In this third part of our ongoing "Countdown Numbers Round Solver" series, I'll be making the solver available for you (and others) on the Internet. This means that you'll be able to visit the solver online and challenge yourself with random problems, or request a solution to a particular problem.

I want to keep costs as low as possible, so I want to serve the solver using [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static), which means we'll need to make sure the computation involved when solving the problem happens on the user's device, not a compute element in Azure (which I'd be paying for).

There are a number of options we could leverage, from any number of JavaScript UI frameworks/libraries or we could use [ASP.NET Core Blazor's](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor) [WebAssembly hosting model](https://learn.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-10.0#blazor-webassembly) as we already have a functioning C# implementation.

Leveraging Blazor would certainly be the quickest "route to market", but let's push ourselves to do something a little outside our comfort zone. We're going to rewrite the iterative deepening variant using the [Rust](https://rust-lang.org/) programming language and leverage the [Yew framework](https://yew.rs/) to deliver the UI experience to our users.

You can find the code associated with this post at [the GitHub repository](https://github.com/newmancodes/yew-countdown-solver) and use the solver itself at [Countdown Solver](https://cds.newman.digital).

## WebAssembly

We need an option that allows the uninformed search computation to happen on the user's device, not in Azure. The best options we have for executing code on the browser are the JavaScript and WebAssembly.

> WebAssembly (abbreviated Wasm) is a binary instruction format for a stack-based virtual machine. Wasm is designed as a portable compilation target for programming languages, enabling deployment on the web for client and server applications.
>
> -- <cite>[WebAssembly.org](https://webassembly.org/)</cite>

I'm not going to be programming this code in the raw binary instruction format that WebAssembly executes! So I need to find a programming language which I can leverage to produce a WASM build artifact that can run on the stacked-based virtual machine (VM) described by Wasm. I've already mentioned C# as a possibility (via Blazor), but there are several options I could choose. I could use C, C++, Go, Python, Java, and many others, but I'm going to be using Rust for this project which has good compilation support for Wasm.

## Rust

Why Rust?

What even is Rust?

The Rust compiler can compile straight to WebAssembly when the wasm32-unknown-unknown target is added by executing the command `rustup target add wasm32-unknown-unknown`.

## Yew + Trunk

### Tailwind CSS

```plaintext
I want to render the target, numbers, and solve button in a nicer way. All the game.board().numbers() should be along side one another. I want them to be black on a white box, with a grey border. If the display is narrow (think mobile in landspace orientation) I want them to split into two rows of three, then three rows of two, then a column. There will be six numbers. The target value should be white on a blue background with a darker grey border. It should be as wide as the numbers beneath it. The solve button should be below all the numbers with some spacing and centrally aligned. I am using tailwind css and I want to use semantic html and be mindful of accessibility standards. Can you make some suggestions for us to try?
```

## Implementing in Rust

Result: No change needed — the linear scan wins
Here's what the data told us:
Approach	1-step	2-step	3-step	4-step	5-step	impossible
Baseline (linear scan)	39 us	507 us	1.11 ms	5.68 ms	24.1 ms	8.20 ms
HashSet mirror	+7%	+6%	+7%	+7%	+9%	+7%
Drop check entirely	-10%	+49%	+41%	+3%	+12%	+25%
Why the linear scan wins here:
- The frontier is small — boards have at most 6 numbers, and iterative deepening limits depth to 1-6. The frontier rarely exceeds a few hundred entries.
- At that size, a sequential scan over a contiguous Vec is cache-friendly and fast. HashSet hashing overhead (computing the hash of a Board on every insert, remove, and lookup) costs more than the scan saves.
- Dropping the check entirely causes frontier explosion — without dedup, duplicate boards flood the frontier, each carrying a cloned StateTraversal parent chain. The cloning cost dominates.

## Playwright

## Deploying to Azure

My Azure setup is fairly simple, I maintain a single Subscription and I tend to deploy various elements into different resource groups for logical separation. I'll be using [Azure Deployment Stacks](https://learn.microsoft.com/en-us/training/modules/introduction-to-deployment-stacks/) for this project so that the only mechanism to make changes to my cloud infrastructure is via a deployment (using [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep)), this prevents [configuration drift](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/management-operational-compliance#monitor-for-configuration-drift), and also so I can easily tear down the infrastructure via an `az stack` command. If you want to learn more about Azure Deployment Stacks, John Savill posted a great [Deployment Stacks Deep Dive](https://youtu.be/d1AE8qLwBYw?si=2LJpwiXTFcFAqUv9) video to YouTube.

### GitHub Actions Deployment of Azure Deployment Stack

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
- Store the id of the service principal associated with the app registration to support future deployments

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
principalId=$(az ad sp create \
    --id $appId \
    --query id -o tsv)

# Create the Resource Group, I've selected westeurope as
# Static Web Apps aren't available everywhere.
az group create \
    --name rg-countdown-solver \
    --location westeurope

# Create an RBAC Role Assignment that grants the Service Principal
# both the Contributor and Azure Deployment Stack Owner roles
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

# Store the service principal's Id so it can be excluded from the
# Deployment Stack's deny settings (allowing this service principal
# the ability to make changes to the infrastructure managed by this
# project's stack).
gh secret set DENY_SETTINGS_EXCLUDED_PRINCIPAL --body "$principalId"
```

### Security

## Conclusion

### AI Declaration

I have made use of multiple Generative AI solutions during this project. During the writing process I have consulted with Anthropic's Claude Sonnet 4.5 model via GitHub Copilot (in Visual Studio Code) and a custom agent in OpenCode to help me construct a good flow to the article and perform a content review. During the coding portion of the project I made use of both GitHub Copilot, Claude Code, and OpenCode. I typically use Anthropic's Claude Sonnet 4.5/6 and Claude Opus 4.6 for most of my Generative AI interactions. I use Claude Code and OpenCode to help me navigate issues with Rust, Yew, and Trunk. Occasionally generating plans and executing those plans (you can find these in the GitHub repo at [./.claude/plans](https://github.com/newmancodes/yew-countdown-solver/blob/main/.claude/plans)) or [./.opencode/plans](https://github.com/newmancodes/yew-countdown-solver/tree/main/.opencode/plans). Beyond these usages I have crafted this project by hand and enjoyed the ride. When these technologies helped me over a particular hurdle, I'll call it out below rather than leaving a bland "I prompted" and leaving it as an exercise to the reader to discover how to do that for themselves. There are a lot of people out there that are quick to comment "skill issue! you're using it wrong", but we're all learning these tools in parallel and I certainly haven't gotten this all figured out myself, let's share what's worked for us and we'll all improve together and figure out where the sharp edges are. Throughout this blog post I'll call out some specific areas where I utilised AI to good effect and I'll add a summary at the end of the post covering where I am with AI-assisted coding at this time.
