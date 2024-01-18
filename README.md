# azdo-oidc

## Service Principal

### Resources

```bash
# Variables

# Azure Subscription Id
AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# Azure DevOps Organization Name
AZDO_ORGANIZATION_NAME="squintelier"
# Azure DevOps Project Name
AZDO_PROJECT_NAME="azdo-oidc"                                  
# Azure DevOps Service Account Name
AZDO_SERVICE_ENDPOINT_NAME="sc-manual-mpn-test"                      

# Azure Subscription Name
result=$(az account show -s $AZURE_SUBSCRIPTION_ID)
AZURE_SUBSCRIPTION_NAME=$(echo $result | jq -r '.name') 

# Azure DevOps base URL
AZDO_BASE_URL="https://dev.azure.com/$AZDO_ORGANIZATION_NAME"  

# Set Azure DevOps defaults
result=$(az devops configure --defaults organization=$AZDO_BASE_URL)

# Create DevOps Project
if ! az devops project list | jq -e --arg PROJECT_NAME "$AZDO_PROJECT_NAME" '.[] | select(.name == $PROJECT_NAME)' > /dev/null; then
  result=$(az devops project create --name $AZDO_PROJECT_NAME --description $AZDO_PROJECT_NAME --visibility private)
fi

# Create App Registration
result=$(az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/$AZURE_SUBSCRIPTION_ID" --name app-$AZDO_ORGANIZATION_NAME-azdo-oidc)

# Get AAD Application Id
AAD_CLIENT_ID=$(echo $result | jq -r '.appId')
AAD_TENANT_ID=$(echo $result | jq -r '.tenant')

# Create Service Endpoint Configuration params file params.azdo.json
cat <<EOF > params.azdo.json
{
  "data": {
    "subscriptionId": "${AZURE_SUBSCRIPTION_ID}",
    "subscriptionName": "$AZURE_SUBSCRIPTION_NAME"
  },
  "authorization": {
    "parameters": {
        "serviceprincipalid": "${AAD_CLIENT_ID}",
        "tenantid": "${AAD_TENANT_ID}"
    },
    "scheme": "WorkloadIdentityFederation"
  },
  "description": "${AZDO_SERVICE_ENDPOINT_NAME}",
  "name": "${AZDO_SERVICE_ENDPOINT_NAME}",
  "serviceEndpointProjectReferences": [
    {
      "description": "${AZDO_SERVICE_ENDPOINT_NAME}",
      "name": "${AZDO_SERVICE_ENDPOINT_NAME}",
      "projectReference": {
        "name": "${AZDO_PROJECT_NAME}"
      }
    }
  ],
  "type": "azurerm",
  "url": "https://management.azure.com/"
}
EOF

# Create Or Get Service Endpoint

if ! az devops service-endpoint list --project $AZDO_PROJECT_NAME | jq -e --arg SE_NAME "$AZDO_SERVICE_ENDPOINT_NAME" '.[] | select(.name == $SE_NAME)' > /dev/null; then
  result=$(az devops service-endpoint create --service-endpoint-configuration params.azdo.json --organization $AZDO_BASE_URL --project $AZDO_PROJECT_NAME --detect true)
  # Service Endpoint Id
  SERVICE_ENDPOINT_ID=$(echo $result | jq -r '.id')                                                                                                                                 
else  
  # Service Endpoint Id
  SERVICE_ENDPOINT_ID=$(az devops service-endpoint list --project $AZDO_PROJECT_NAME | jq -r --arg NAME "$AZDO_SERVICE_ENDPOINT_NAME" '.[] | select(.name==$NAME) | .id')
  result=$(az devops service-endpoint show --project $AZDO_PROJECT_NAME --id $SERVICE_ENDPOINT_ID)
fi

# Service Endpoint Issuer
SERVICE_ENDPOINT_ISSUER=$(echo $result | jq -r '.authorization.parameters.workloadIdentityFederationIssuer')                                                                      
# Service Endpoint Subject
SERVICE_ENDPOINT_SUBJECT=$(echo $result | jq -r '.authorization.parameters.workloadIdentityFederationSubject')                                                                    


# Create Federated Credential Configuration params file params.json
PARAMS_NAME="$AZDO_PROJECT_NAME-federated-identity"
PARAMS_ISSUER="${SERVICE_ENDPOINT_ISSUER}"
PARAMS_SUBJECT="${SERVICE_ENDPOINT_SUBJECT}"
PARAMS_DESCRIPTION="Federation for Service Connection $AZDO_SERVICE_ENDPOINT_NAME in $AZDO_BASE_URL/$AZDO_PROJECT_NAME/_settings/adminservices?resourceId=$SERVICE_ENDPOINT_ID"

cat <<EOF > params.create.json
{
  "name": "${PARAMS_NAME}",
  "issuer": "${SERVICE_ENDPOINT_ISSUER}",
  "subject": "${PARAMS_SUBJECT}",
  "description": "${PARAMS_DESCRIPTION}",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF

cat <<EOF > params.update.json
{
  "issuer": "${SERVICE_ENDPOINT_ISSUER}",
  "subject": "${PARAMS_SUBJECT}",
  "description": "${PARAMS_DESCRIPTION}",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF

# Create Or Update Federated Credential
if ! az ad app federated-credential list --id $AAD_CLIENT_ID | jq -e --arg NAME "$PARAMS_NAME" '.[] | select(.name == $NAME)' > /dev/null; then
  result=$(az ad app federated-credential create --id $AAD_CLIENT_ID --parameters params.create.json) 
else
  FEDERATED_CREDENTIAL_ID=$(az ad app federated-credential list --id $AAD_CLIENT_ID | jq -r -e --arg NAME "$PARAMS_NAME" '.[] | select(.name == $NAME) | .id')
  result=$(az ad app federated-credential update --id $AAD_CLIENT_ID --federated-credential-id $FEDERATED_CREDENTIAL_ID --parameters params.update.json) 
fi

```
### pipeline

Git Clone Project if necessary
```bash

git clone https://$AZDO_ORGANIZATION_NAME@dev.azure.com/$AZDO_ORGANIZATION_NAME/$AZDO_PROJECT_NAME/_git/$AZDO_PROJECT_NAME
```

Go to the project
```bash
cd $AZDO_PROJECT_NAME

PIPELINE_DIR="pipelines"
IDENTITY_TYPE="sp"

# Create directory if not exists
if [ ! -d "$PIPELINE_DIR" ]; then
    mkdir $PIPELINE_DIR
fi

# Store pipeline
cat <<EOF > $PIPELINE_DIR/$IDENTITY_TYPE.yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: '${AZDO_SERVICE_ENDPOINT_NAME}'
    scriptType: 'pscore'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az account show --query id -o tsv
EOF

# Commit pipeline
git config --global user.name $GIT_USER
git config --global user.email $GIT_EMAIL

if ! git diff --quiet HEAD -- "./$PIPELINE_DIR/$IDENTITY_TYPE.yaml" ; then
  git add ./$PIPELINE_DIR/$IDENTITY_TYPE.yaml
  git commit -m "📊 Add or Update pipeline."
  git push origin main
else
  echo "Nothing to commit."
fi
```

Create pipeline
```bash 
az devops configure --defaults organization=$AZDO_BASE_URL

az pipelines create \
    --name "$IDENTITY_TYPE-pipeline" \
    --description "This is a sample pipeline to use federated identity" \
    --repository $AZDO_PROJECT_NAME \
    --repository-type tfsgit \
    --branch main \
    --yaml-path $PIPELINE_DIR/$IDENTITY_TYPE.yaml \
    --project $AZDO_PROJECT_NAME

```


## User Assigned Managed Identity

### Resources
```bash
# Variables

GIT_USER="Sujith Quintelier"
GIT_EMAIL="squintelier@company.com"

# Azure Subscription Id
AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  
# Azure location
AZURE_LOCATION="westeurope"                                   
# Azure DevOps Project Name
AZDO_PROJECT_NAME="azdo-oidc"                                  
# Azure DevOps Organization Name
AZDO_ORGANIZATION_NAME="squintelier"                           
# Azure DevOps Service Account Name
AZDO_SERVICE_ENDPOINT_NAME="sc-manual-mpn-test-mi"                      

# Azure Resource Group
AZURE_RG="rg-$AZDO_PROJECT_NAME"                                         
# Managed Identity Name
ID_NAME="id-$AZDO_PROJECT_NAME"                                          
# squintelier|xpirit
AZDO_BASE_URL="https://dev.azure.com/$AZDO_ORGANIZATION_NAME"  # Azure DevOps base URL
# Azure Subscription Name
result=$(az account show -s $AZURE_SUBSCRIPTION_ID)
AZURE_SUBSCRIPTION_NAME=$(echo $result | jq -r '.name') 

# Set Azure DevOps defaults
result=$(az devops configure --defaults organization=$AZDO_BASE_URL)

# Create DevOps Project
if ! az devops project list | jq -e --arg PROJECT_NAME "$AZDO_PROJECT_NAME" '.value[] | select(.name == $PROJECT_NAME)' > /dev/null; then
  result=$(az devops project create --name $AZDO_PROJECT_NAME --description $AZDO_PROJECT_NAME --visibility private)
fi

# Create Resource Group
result=$(az group create --location $AZURE_LOCATION --name $AZURE_RG)

# Create Managed Identity
result=$(az identity create --name $ID_NAME --resource-group $AZURE_RG)

# Managed Identity Id
MI_ID=$(echo $result | jq -r '.id')                       
# AAD Application Id
AAD_CLIENT_ID=$(echo $result | jq -r '.clientId')         
# AAD Principal Id
AAD_PRINICIPAL_ID=$(echo $result | jq -r '.principalId')  
# AAD Tenant Id
AAD_TENANT_ID=$(echo $result | jq -r '.tenantId')         

# Role Assignment
#
# With Graph Permissions (uncomment below)
# az role assignment create --role "Contributor" --assignee $MI_ID --scope /subscriptions/$AZURE_SUBSCRIPTION_ID
# 
# Without Graph Permissions (uncomment below)
az role assignment create --role "Contributor" --assignee-object-id $AAD_PRINICIPAL_ID --assignee-principal-type ServicePrincipal --scope /subscriptions/$AZURE_SUBSCRIPTION_ID

# Create Service Endpoint Configuration params file params.azdo.json
cat <<EOF > params.azdo.json
{
  "data": {
    "subscriptionId": "${AZURE_SUBSCRIPTION_ID}",
    "subscriptionName": "$AZURE_SUBSCRIPTION_NAME"
  },
  "authorization": {
    "parameters": {
        "serviceprincipalid": "${AAD_CLIENT_ID}",
        "tenantid": "${AAD_TENANT_ID}"
    },
    "scheme": "WorkloadIdentityFederation"
  },
  "description": "${AZDO_SERVICE_ENDPOINT_NAME}",
  "name": "${AZDO_SERVICE_ENDPOINT_NAME}",
  "serviceEndpointProjectReferences": [
    {
      "description": "${AZDO_SERVICE_ENDPOINT_NAME}",
      "name": "${AZDO_SERVICE_ENDPOINT_NAME}",
      "projectReference": {
        "name": "${AZDO_PROJECT_NAME}"
      }
    }
  ],
  "type": "azurerm",
  "url": "https://management.azure.com/"
}
EOF

# Create Or Get Service Endpoint

if ! az devops service-endpoint list --project $AZDO_PROJECT_NAME | jq -e --arg SE_NAME "$AZDO_SERVICE_ENDPOINT_NAME" '.[] | select(.name == $SE_NAME)' > /dev/null; then
  result=$(az devops service-endpoint create --service-endpoint-configuration params.azdo.json --organization $AZDO_BASE_URL --project $AZDO_PROJECT_NAME --detect true)
  # Service Endpoint Id
  SERVICE_ENDPOINT_ID=$(echo $result | jq -r '.id')                                                                                                                                 
else  
  # Service Endpoint Id
  SERVICE_ENDPOINT_ID=$(az devops service-endpoint list --project $AZDO_PROJECT_NAME | jq -r --arg NAME "$AZDO_SERVICE_ENDPOINT_NAME" '.[] | select(.name==$NAME) | .id')
  result=$(az devops service-endpoint show --project $AZDO_PROJECT_NAME --id $SERVICE_ENDPOINT_ID)
fi

# Service Endpoint Issuer
SERVICE_ENDPOINT_ISSUER=$(echo $result | jq -r '.authorization.parameters.workloadIdentityFederationIssuer')                                                                      
# Service Endpoint Subject
SERVICE_ENDPOINT_SUBJECT=$(echo $result | jq -r '.authorization.parameters.workloadIdentityFederationSubject')                                                                    

# Create Federated Credential Configuration params file params.json
PARAMS_NAME="$AZDO_PROJECT_NAME-federated-identity"
PARAMS_ISSUER="${SERVICE_ENDPOINT_ISSUER}"
PARAMS_SUBJECT="${SERVICE_ENDPOINT_SUBJECT}"
PARAMS_DESCRIPTION="Federation for Service Connection $AZDO_SERVICE_ENDPOINT_NAME in $AZDO_BASE_URL/$AZDO_PROJECT_NAME/_settings/adminservices?resourceId=$SERVICE_ENDPOINT_ID"

# Create Or Update Federated Credential
if ! az identity federated-credential list --identity-name $ID_NAME --resource-group $AZURE_RG | jq -e --arg NAME "$PARAMS_NAME" '.[] | select(.name == $NAME)' > /dev/null; then
  result=$(az identity federated-credential create --identity-name $ID_NAME --name $PARAMS_NAME  --resource-group $AZURE_RG --audiences "api://AzureADTokenExchange" --issuer $SERVICE_ENDPOINT_ISSUER --subject $PARAMS_SUBJECT)
else
  result=$(az identity federated-credential update --identity-name $ID_NAME --name $PARAMS_NAME  --resource-group $AZURE_RG --audiences "api://AzureADTokenExchange" --issuer $SERVICE_ENDPOINT_ISSUER --subject $PARAMS_SUBJECT)
fi

```
### pipeline

Git Clone Project if necessary
```bash

git clone https://$AZDO_ORGANIZATION_NAME@dev.azure.com/$AZDO_ORGANIZATION_NAME/$AZDO_PROJECT_NAME/_git/$AZDO_PROJECT_NAME
```

Go to the project
```bash
cd $AZDO_PROJECT_NAME

PIPELINE_DIR="pipelines"
IDENTITY_TYPE="mi"

# Create directory if not exists
if [ ! -d "$PIPELINE_DIR" ]; then
    mkdir $PIPELINE_DIR
fi

# Store pipeline
cat <<EOF > $PIPELINE_DIR/$IDENTITY_TYPE.yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: '${AZDO_SERVICE_ENDPOINT_NAME}'
    scriptType: 'pscore'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az account show --query id -o tsv
EOF

# Commit pipeline
git config --global user.name $GIT_USER
git config --global user.email $GIT_EMAIL

if ! git diff --quiet HEAD -- "./$PIPELINE_DIR/$IDENTITY_TYPE.yaml" ; then
  git add ./$PIPELINE_DIR/$IDENTITY_TYPE.yaml
  git commit -m "📊 Add or Update pipeline."
  git push origin main
else
  echo "Nothing to commit."
fi
```

Create pipeline
```bash 
az devops configure --defaults organization=$AZDO_BASE_URL

az pipelines create \
    --name "$IDENTITY_TYPE-pipeline" \
    --description "This is a sample pipeline to use federated identity" \
    --repository $AZDO_PROJECT_NAME \
    --repository-type tfsgit \
    --branch main \
    --yaml-path $PIPELINE_DIR/$IDENTITY_TYPE.yaml \
    --project $AZDO_PROJECT_NAME

```
