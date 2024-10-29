# SKIP 002 - SpinKube Azure

Summary: A repository for all things Azure related in the SpinKube project

Owner: Jiaxiao (Joe) Zhou <jiazho@microsoft.com>

Impacted Projects:

- [ ] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [x] Creates a new project

Created: June 4th, 2024

Updated: June 4th, 2024

## Background

I have worked on a project called [spinkube-azure](https://github.com/Mossaka/spinkube-azure) which essentially is a helm chart that deploys spinkube operator and its related components on Azure Kubernetes Service (AKS) cluster, with a focus on the Azure specific configurations and integrations. The goal of this project is to provide a one-stop shop for all things Azure related and to make it the eaisest way to deploy spinkube on AKS. Features may include:

- AppInsights integration
- Workload Identity integration
- Azure Monitor integration
- etc.

It shares a lot of similarities with the [spinkube-oneclick](https://github.com/jpflueger/spinkube-oneclick) project, which is also not in the SpinKube organization.

The intended usage of the project is twofold:

1. Provide an open-source Helm chart that users can easily install on their AKS clusters.
2. Package the Helm chart as an artifact for the Azure Marketplace, or as a Microsoft managed service such as AKS Extension.

## Proposal

I am proposing to create a new repository under the SpinKube organization called `azure` to host the spinkube-azure project. This repository will be the home for all things Azure related in the SpinKube project. This will allow us to have a dedicated place to work on Azure specific features and configurations, and to make it easier for the community to contribute to the project.

Specially I am proposing to move the [spinkube-azure](https://github.com/Mossaka/spinkube-azure) project to the new repository and rename it to `azure`. I will be the owner of the repository and will be responsible for maintaining it.

A roadmap for the project will be created to outline the features and integrations that we want to add to the project. This will be a living document that will be updated as we make progress on the project.

## Alternatives Considered
