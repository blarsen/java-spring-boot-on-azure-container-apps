# Java Spring Boot On Azure Container Apps (jsboaca)
This repository contains a minimal Java Spring Boot app with the scaffolding needed to build and deploy it on Azure Container Apps. This is acheived in the form of a fully automated CI/CD pipeline based on Github Actions.

The pipeline

- performs linting of the project
- builds the project using Maven
- builds a container image
- performs security scanning of the image
- uploads the image to an Azure container registry
- deploys the image to Azure Container Apps

Below you will find documentation on what needs to be done on Azure and Github respectively to get it up and running.

# Prerequisites

## Software

In order to prepare the pipeline and deploy through it you will need the following tools installed on your development system:

1. __The Azure CLI (az)__  
This can be installed...

2. __The GitHub CLI (gh)__  
asdasda

## Azure

You will need an Azure account with at least the Contributor role in the subscription in which you plan to install the container app. __In addition one of the commands below will need to run from an account with the Owner role in the subscription.__ This is a once off creation of a service pricipal with an associated secret that is only needed in the initial setup of the CI/CD pipeline.

# Manual steps

## Azure setup

__1. Initial setup__  
Make sure that you are logged in to Azure with the correct account:
```
$ az login
```
As we proceed you will need to know the ID of the subscription that you are logged in to. If you do not already know this you can figure it out by running
```
$ az account show --query id
```

__2. Create an Azure resource group__  
First decide in what location you want the resource group to reside and what the resource group should be named. In this example we are using the location _norwayeast_ and the resource group name _jsboacarg_.
```
$ az group create -l norwayeast -n jsboacarg
```
__3. Create an Azure container registry__  
First decide what the container registry should be named. In this example we use the name _jsboacacr_. We are also specifying that this should be a basic container registry, and that it should reside in the resource group we just created.
```
$ az acr create -g jsboacarg -n jsboacacr --sku Basic
```

__4. Create an Azure container app__  
Next, we create the container app that we should deploy to. 

Before we proceed, if you haven't already done so, install the Azure Container Apps extension for the CLI.
```
$ az extension add --name containerapp --upgrade
```
Then create an environment for the container app to run in
```
$ az containerapp env create -g jsboacarg --location norwayeast -n jsboacaenv
```
Finally, create an (empty) container app. Note that there are TONS of parameters that can be specified here or tweaked later, tuning things like logging, CPUs, memory, ports, scaling etc. In this example we create the app with all the default settings.
```
$ az containerapp create -g jsboacarg --env jsboacaenv -n jsboaca 
```
__5. Create an Azure service principal__  

GitHub Action workflow needs to login to Azure, and uses the service principal for that.

__NOTE: This step must be performed by an account that has the Owner role in the Azure subscription that you are working on__  

First decide what the service principal should be named. In this example we use the name _jsboacasp_.
You will also have to specify the ID of the subscription you are using (here we do it by command substitution) as well as the resource group.
```
$ az ad sp create-for-rbac \
    --name jsboacasp \
    --role contributor \
    --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/jsboacarg \
    --sdk-auth
```
__NOTE: Make sure that you save the output from this command! You will need it later and the clientSecret is not obtainable by any other means.__  
  
This is sample output from the create-for-rbac command
```
 {
  "clientId": "0d96ee1e-7646-43d6-b835-20bd634bb267",
  "clientSecret": "ung8Q~lok-rwsO_P_hCz_6LlHop6pjUU3xePyap.",
  "subscriptionId": "55fca7d3-2956-3d09-8b58-4326245b6aae",
  "tenantId": "22ca942f-06c2-4f38-9407-0e447fefbb81",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```
 
 
## GitHub setup

(add login)

First you need to create a GitHub secret containing the result of the create-for-rbac command you executed earlier.
```
$ gh secret set JSPOACA_AZURE_CREDENTIALS <<EOD
 {
  "clientId": "0d96ee1e-7646-43d6-b835-20bd634bb267",
  "clientSecret": "ung8Q~lok-rwsO_P_hCz_6LlHop6pjUU3xePyap.",
  "subscriptionId": "55fca7d3-2956-3d09-8b58-4326245b6aae",
  "tenantId": "22ca942f-06c2-4f38-9407-0e447fefbb81",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
EOD
```


Next you need to set up two secrets containing the Azure container registry and username. The needed information can be obtained with these Azure commands:
```
$ az acr update -n jsboacacr --admin-enabled true
$ az acr credential show -n jsboacacr
```
Example output from the last command:
```
{
  "passwords": [
    {
      "name": "password",
      "value": "Ye/=HJEl==U1xXmEm11QKZ36c194v8Ex"
    },
    {
      "name": "password2",
      "value": "by2H=nK6Yz+pDV3V7rAQoXbazHu/TW6u"
    }
  ],
  "username": "jsboacacr"
}
```

Use the password value and username to set the two secrets below.

```
$ gh secret set JSBOACA_REGISTRY_USERNAME --body "jspoacacr"
$ gh secret set JSBOACA_REGISTRY_PASSWORD --body "Ye=HJEl==U1xXmEm11QKZ36c194v8Ex"
```

Ensure that all three secrets actually have been created
```
$ gh secret list
JSBOACA_REGISTRY_PASSWORD  Updated 2022-11-02
JSBOACA_REGISTRY_USERNAME  Updated 2022-11-02
JSPOACA_AZURE_CREDENTIALS  Updated 2022-11-02
```

## Workflow setup

With the necessary preparation in place on Azure and GitHub respectively it is time to update the GitHub Actions workflow to reflect your setup, if you have chosen to name the created resources differently from what is used in this documentation.

Edit .github/workflows/build.yml and make the needed changes. Change occurences of jsboaca* to whatever you chose when you set things up.


