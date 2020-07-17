# Hugo Static Templates with OpenFaaS

This repo contains the Hugo templates and components to run a static website using openfaas

## Key features

* Driven static web pages from FaaS
* Templates scaffolded application

## Possible Improvements
* Disaster Recovery
* MakeFile

## Pre-requisites

1. faas-cli
1. OpenFaaS with TLS on K8's -> check [patrickguyrodies/openfaas](https://patrickguyrodies@bitbucket.org/patrickguyrodies/openfaas.git)
1. Domain name and TLS certificate
1. Azure Container Registry - We attached the ACR inside our cluster
                $ az aks update -n <cluster_name> -g <resource_group> --attach-acr <registry_name>

### Clone repo and get faas template
Clone the [patrickguyrodies/staticevent](bitbucket.org:patrickguyrodies/staticevent.git) repo and cd into the root of the repo.
>Either create a new branch such as git checkout -b <new-branch> or use the development branch

1. Pull in my †emplate using its URL

                $ faas template pull https://github.com/matipan/openfaas-hugo-template

1. Now create a Hugo site called `pgr095`

                $ faas new --lang hugo pgr095

This will create a folder called pgr095 where we will place the content for the site along with its configuration and any themes we may want.

The pgr095.yml file is called a stack file and can be used to configure the deployment on OpenFaaS.

The following steps are based upon the Hugo [quick-start guide](https://gohugo.io/getting-started/quick-start/#step-2-create-a-new-site):

