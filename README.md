# Hugo Static Templates with OpenFaaS

This repo contains Yaml file to install OpenFaaS with TLS. Certificate is from LetsEncrypt, Domain used is from freenom and DNS from Azure

## Key features

* LetsEncrypt certificate
* DNS Azure service
* Domain name from Free service called freenom
* No use of Helm - :-)


## Possible Improvements
* Disaster Recovery
* MakeFile

## Pre-requisites

1. A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled

1. The kubectl command-line tool installed on your local machine and configured to connect to your cluster. You can read more about installing kubectl in the official documentation.

1. The wget command-line utility installed on your local machine. You can install wget using the package manager built into your operating system.
Once you have these components set up, you’re ready to begin with this guide.

### Creating Domain name