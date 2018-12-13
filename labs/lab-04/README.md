
# Lab 4 - Exploring Mixed Workloads in a kubernetes cluster

## Part 1 - Creating the cluster

Prereqs:

- Azure subscription
- install azure-cli
- install [ace-engine](https://github.com/Azure/acs-engine/releases/latest)
- ssh keys


### Creating the cluster with acs-engine

Acs-engine will allow us to create our kubernetes cluster with windows nodes. Follow this [walkthrough](https://github.com/Azure/acs-engine/blob/master/docs/kubernetes/windows.md) to build a kubernetes cluster and deploy a Winodws web server on it.

## Part 2 - Logging and monitoring

Logging options:

- write to console
- write to file

Use FluentD

## Part 3 - Deploy an Ingress controller