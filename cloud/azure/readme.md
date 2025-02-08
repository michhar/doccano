# Deploy Doccano to Azure

[Doccano](https://github.com/doccano/doccano) is an easy-to-use open-source application for labeling and annotating vision and text data for machine learning experiments.  Below you will find the instructions for deploying Doccano as a Web App to Azure.

## Prerequisites

1. Azure Subscription
2. Azure CLI
3. Resource Group created in Azure
4. Azure Container Registry in the Resource Group

Log in to your subscription with the CLI:
`az login`.  Take note of your Resource Group and ACR name.

## Build the Docker image locally and push to your Azure Container Registry

- Based on [az acr build in API reference docs](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-build).

Build the docker image on your command line (here, using a unix command line) and push it to ACR this Docker build uses `backend/requirements.txt` for the Python package installs. Run the following in the base of the repo.

`az acr build --file docker/Dockerfile.piponly --registry <your ACR name> --platform linux --timeout 3600 --image doccano-app .`

- Fill in `<your ACR name>` with the name of your provisioned ACR (note, you must be logged in to Azure through the CLI and have chosen the correct subscription to use wherein the ACR has been provisioned).
- It is a good idea to test the Docker image locally by running the container and logging in to the Doccano app (see [repo](https://github.com/doccano/doccano?tab=readme-ov-file#docker) on how to do this).
- If this doesn't work, then build the Docker image manually, with your local install of Docker, and push that image to your ACR as in [this section](https://learn.microsoft.com/en-us/azure/app-service/tutorial-custom-container?tabs=azure-cli&pivots=container-linux#iii-push-the-sample-image-to-azure-container-registry).

## Steps to set up and deploy web app which will use a Docker conatiner

- Based on [Migrate custom software to Azure App Service using a custom container](https://learn.microsoft.com/en-us/azure/app-service/tutorial-custom-container?tabs=azure-cli&pivots=container-linux).
- Fill in the following placeholders with your specific values:
  - `<your RG name>`
  - `<your ACR name>`

Create a managed identity in the resource group:

`az identity create --name apps-managed-id --resource-group <your RG name>`

Retrieve the principal ID for the managed identity:

`principalId=$(az identity show --resource-group <your RG name> --name apps-managed-id --query principalId --output tsv)`

Retrieve the resource ID for the container registry:

`registryId=$(az acr show --resource-group <your RG name> --name <your ACR name> --query id --output tsv)`

Grant the managed identity permission to access the container registry:

` az role assignment create --assignee $principalId --scope $registryId --role "AcrPull"`

Create an App Service plan (if you do not have this already) using the az appservice plan create command:

`az appservice plan create --name myAppServicePlan --resource-group <your RG name> --is-linux`

Create the web app with the az webapp create command:

`az webapp create --resource-group <your RG name> --plan myAppServicePlan --name doccano-labeling-app --deployment-container-image-name <your ACR name>.azurecr.io/doccano-app:latest`

Use az webapp config appsettings set to set the WEBSITES_PORT, ADMIN_USERNAME, ADMIN_PASSWORD and ADMIN_EMAIL environment variable as expected by the app code:

`az webapp config appsettings set --resource-group <your RG name> --name doccano-labeling-app --settings WEBSITES_PORT=8000`

`az webapp config appsettings set --resource-group <your RG name> --name doccano-labeling-app --settings ADMIN_USERNAME=<admin username>`

`az webapp config appsettings set --resource-group <your RG name> --name doccano-labeling-app --settings ADMIN_PASSWORD=<unique admin password>`
> Important: please take note of this password as you may not get a chance to view again!

`az webapp config appsettings set --resource-group <your RG name> --name doccano-labeling-app --settings ADMIN_EMAIL=<an admin email>`

Enable the user-assigned managed identity in the web app with the az webapp identity assign command:

`id=$(az identity show --resource-group <your RG name> --name apps-managed-id --query id --output tsv)`

`az webapp identity assign --resource-group <your RG name> --name doccano-labeling-app --identities $id`

Configure your app to pull from Azure Container Registry by using managed identities.

`appConfig=$(az webapp config show --resource-group <your RG name> --name doccano-labeling-app --query id --output tsv)`

`az resource update --ids $appConfig --set properties.acrUseManagedIdentityCreds=True`

Set the client ID your web app uses to pull from Azure Container Registry. This step isn't needed if you use the system-assigned managed identity.

`clientId=$(az identity show --resource-group <your RG name> --name apps-managed-id --query clientId --output tsv)`

`az resource update --ids $appConfig --set properties.AcrUserManagedIdentityID=$clientId`

Enable CI/CD in App Service:

`cicdUrl=$(az webapp deployment container config --enable-cd true --name doccano-labeling-app --resource-group <your RG name> --query CI_CD_URL --output tsv)`

Create a webhook in your container registry using the CI_CD_URL you got from the last step:

`az acr webhook create --name appserviceCD --registry <your ACR name> --uri $cicdUrl --actions push --scope doccano-app:latest`

`eventId=$(az acr webhook ping --name appserviceCD --registry <your ACR name> --query id --output tsv)`

`az acr webhook list-events --name appserviceCD --registry <your ACR name> --query "[?id=='$eventId'].eventResponseMessage"`

To test the app, browse to https://doccano-labeling-app.azurewebsites.net and log in with your admin user and password specified above.


## Troubleshooting

- Please refer to [Azure App Service on Linux FAQ](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/faqs-app-service-linux#built-in-images) for help or open a support ticket in Azure.