# Hugo Static Templates with OpenFaaS

This repo contains the Hugo templates and components to run a static website using openfaas

## Key features

* Driven static web pages from FaaS
* Templates scaffolded application

## Possible Improvements
* Vault for openfaas login
* Disaster Recovery
* MakeFile

## Pre-requisites

1. faas-cli
1. OpenFaaS with TLS on K8's -> check [patrickguyrodies/openfaas](https://patrickguyrodies@bitbucket.org/patrickguyrodies/openfaas.git)
1. Domain name and TLS certificate
1. Azure Container Registry - We attached the ACR inside our cluster

                $ az aks update -n <cluster_name> -g <resource_group> --attach-acr <registry_name>
1. OpenFaaS FunctionIngress CRD, please check  https://github.com/openfaas-incubator/ingress-operator.git. Added as submodule.

### Clone repo and get faas template
Clone the [patrickguyrodies/staticevent](bitbucket.org:patrickguyrodies/staticevent.git) repo and cd into the root of the repo.
>Either create a new branch such as git checkout -b <new-branch> or use the development branch

1. Pull in my template using its URL

                $ faas template pull https://github.com/matipan/openfaas-hugo-template

1. Now create a Hugo site called `pgr095`

                $ faas new --lang hugo pgr095

This will create a folder called pgr095 where we will place the content for the site along with its configuration and any themes we may want.

The pgr095.yml file is called a stack file and can be used to configure the deployment on OpenFaaS.

The following steps are based upon the Hugo [quick-start guide](https://gohugo.io/getting-started/quick-start/#step-2-create-a-new-site):

                $ cd example-site

                # Create a new site
                $ hugo new site .

                # Add a custom theme

                $ git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke

                # Append the theme to the config file
                $ echo 'theme = "ananke"' >> config.toml

When you are developing new content you’ll probably want to see what it looks like before deploying it. To do this, you can use the hugo server command inside the function’s directory:

                $hugo server
                Watching for changes in /home/capitan/src/gitlab.com/matipan/openfaas-hugo-blog/blog/{archetypes,content,static}
                Watching for config changes in /home/capitan/src/gitlab.com/matipan/openfaas-hugo-blog/blog/config.toml
                Environment: "development"
                Serving pages from memory
                Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
                Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
                Press Ctrl+C to stop

Now go to http://localhost:1313 and check out what your site looks like.

Edit to config.toml and set the baseURL to the custom DNS domain that you will be using.

1. Check your OpenFaaS Yaml stack file to have correct settings

                $ nano pgr095.yml

1. Login to your repo hub, Docker or Azure repo

                $ docker login pgr095.azurecr.io -- replace with your docker hub url

1. Login to OpenFaaS interface - Create a pass txt file with your password

                $ cat ~/faas_pass.txt | faas-cli login -u admin --password-stdin --gateway https://openfaas.pgr095.tk

Now deploy your site: You can also run faas-cli build, push & deploy if you want to separate the calls

                $ export OPENFAAS_URL="www.pgr095.tk" # Set when installing OpenFaaS

                $ faas-cli up -f pgr095.yml

1. Map the custom domain with a FunctionIngress
Now it’s time to map our Hugo blog to the custom domain above. We can do that with a FunctionIngress record. Check hugo_ingress.yaml

                $ nano hugo_ingress.yaml

Now apply the file with 

                $ kubectl apply -f hugo_ingress.yaml

1. Check your implementation

    1. Check your new ingress:

                $ kubectl get ingress -n openfaas
    
    1. Check the certificate:

                $ kubectl get cert -n openfaas

Navigate to your domain and check out your new site.

