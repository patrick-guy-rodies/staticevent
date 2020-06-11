# Hugo Deployment using OpenFaaS, K8s and Helm

This repo contains all the code required to deploy & configure Hugo site using OpenFaaS


## Key features

* Hugo template based site
* A managed Azure AKS cluster


## CI/CD
The repo is using GitHUB action for CI/CD at the moment.



### Pre-requisites

1. terraform

1. helm

1. az (Azure CLI)

1. kubectl

### Deploying

1. Clone the [patrickguyrodies/staticevent](bitbucket.org:patrickguyrodies/staticevent.git) repo and cd into the root of the repo.


1. To deploy Hugo  use the following command. (Make sure your replace the <ENVIRONMENT_NAME> with your value):

```shell
make apply -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

## Destroying a Deployment

Note the following steps will delete a Vault Server, including all other services deployed as part of the pipeline

### Steps to Destroy a Vault Server deployed via Drone

1. Switch to the branch of the vault environment you wish to destroy.

1. To destroy the environment create a new commit with message "DESTROY" and push to the remote branch. This will trigger the Vault deletion pipeline in Drone.

```shell
git commit --allow-empty -m "DESTROY"
git push origin <YOUR_BRANCH>
```

### Steps to Destroy hugo deployed via your Local Machine

To destroy Hashicorp Vault use the following command. (Make sure your replace the <ENVIRONMENT_NAME> with your value.):

```shell
make destroy -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```