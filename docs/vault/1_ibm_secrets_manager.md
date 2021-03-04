# Lab1: Manage Encrypted Secrets with IBM Secrets Manager (Managed Vault)

Suppose you have an application `guestbook`, which persists data to a MongoDB instance. To access MongoDB, `guestbook` needs a username and password for MongoDB. Instead of configuring credentials as code64 encoded secrets, we store encrypted credentials in Vault. Using the `Vault Agent Injector` the app retrieves the encrypted secrets from Vault using Kubernetes authentication and an in-memory volume mount.

In this lab we will create and use a free instance of [IBM Secrets Manager](https://cloud.ibm.com/docs/secrets-manager) with `Lite` plan. The free `Lite` plan is limited to 1 instance per account. IBM Secrets Manager is a cloud managed instance of Vault.

Steps:

1. Create an IBM Secrets Manager Instance,
1. Create an IBM Cloud API Key,
1. Create an Access Token,
1. Using the Vault CLI,
1. Create a Secret Group,
1. Static versus Dynamic Secrets,
1. Create a Static Secret,
1. Read Secret,

Make sure you are logged in to your IBM Cloud account,

```bash
ibmcloud login [-sso]
```

Check for existing resources of IBM Secrets Manager,

```bash
ibmcloud resource service-instances
```

If you already have a `Lite` instance of IBM Secrets Manager, you can skip step 1 to create a new instance, or you can delete the existing instance as follows using the name or id of the service,

```bash
SERVICE_NAME=remkohdev-vault-1
SERVICE_ID=crn:v1:bluemix:public:secrets-manager:us-south:a/3fe3c0de197257ef62d81c9f9c0f33aa:5c9c312c-1d3f-4cc7-839b-a95f7305896f::

ibmcloud resource service-instance-delete $SERVICE_NAME
```

## Create an IBM Secrets Manager Instance

Make sure the `secrets-manager` plugin for IBM Cloud CLI is installed,

```bash
$ ibmcloud plugin list

Listing installed plug-ins...

Plugin Name                                 Version   Status             Private endpoints supported   
cloud-object-storage                        1.2.2                        false   
machine-learning                            3.0.2                        false   
analytics-engine                            1.0.166                      false   
code-engine/ce                              0.5.20                       false   
container-registry                          0.1.514                      false   
power-iaas                                  0.3.1                        false   
cloud-databases                             0.10.2                       false   
cloud-dns-services                          0.3.4                        false   
cloud-functions/wsk/functions/fn            1.0.49                       false   
cloud-internet-services                     1.13.1                       false   
push-notifications                          1.0.3                        false   
app-configuration                           0.0.1                        false   
dbaas-cli                                   1.7.1                        false   
doi                                         0.3.1                        false   
secrets-manager                             0.0.6                        false   
tg-cli/tg                                   0.3.0                        false   
schematics                                  1.5.1                        false   
watson                                      0.0.9                        false   
catalogs-management                         1.0.6                        true   
hpvs                                        1.4.2                        false   
observe-service/ob                          1.0.61                       false   
tke                                         1.1.4                        false   
activity-tracker                            3.3.4                        false   
auto-scaling                                0.2.8                        false   
container-service/kubernetes-service        1.0.233                      false   
dl-cli                                      0.3.0                        false   
event-streams                               2.3.0                        false   
key-protect                                 0.6.0                        false   
vpc-infrastructure/infrastructure-service   0.7.8     Update Available   false   
whcs                                        0.0.5                        false 
```

If the `secrets-manager` plugin is not installed, install it,

```bash
ibmcloud plugin install secrets-manager
```

With the IBM Cloud CLI and `secrets-manager` plugin installed, create a free `Lite` instance of IBM Secrets Manager with name set in the environment variable `SM_NAME`,

```bash
SM_NAME=remkohdev-vault-1
SM_PLAN=lite
REGION=us-south
RG=default

ibmcloud resource service-instance-create $SM_NAME secrets-manager $SM_PLAN $REGION -g $RG
```

To view existing resource groups,

```bash
$ ibmcloud resource groups

Retrieving all resource groups under account 3fe3c0de197257ef62d81c9f9c0f33aa as remkohdev@us.ibm.com...
OK
Name      ID                                 Default Group   State   
default   46ca8ccd59ae4f97b2a6b84539342d1c   true            ACTIVE 
```

To view available regions,

```bash
$ ibmcloud regions
Listing regions...

Name            Display name   
au-syd          Sydney   
in-che          Chennai   
jp-osa          Osaka   
jp-tok          Tokyo   
kr-seo          Seoul   
eu-de           Frankfurt   
eu-gb           London   
ca-tor          Toronto   
us-south        Dallas   
us-south-test   Dallas Test   
us-east         Washington DC   
br-sao          Sao Paolo 
```

Note that not all services might be available in all regions. For this lab, I used the `us-south` region.

```bash
$ ibmcloud resource service-instance $SM_NAME

Retrieving service instance remkohdev-vault-1 in all resource groups under account REMKO DE KNIKKER's Account as remkohdev@us.ibm.com...
OK
                       
Name:                  remkohdev-vault-1   
ID:                    crn:v1:bluemix:public:secrets-manager:us-south:a/3fe3c0de197257ef62d81c9f9c0f33aa:5c9c312c-1d3f-4cc7-839b-a95f7305896f::   
GUID:                  5c9c312c-1d3f-4cc7-839b-a95f7305896f   
Location:              us-south   
Service Name:          secrets-manager   
Service Plan Name:     lite   
Resource Group Name:   default   
State:                 active   
Type:                  service_instance   
Sub Type:                 
Created at:            2021-02-25T15:41:12Z   
Created by:            remkohdev@us.ibm.com   
Updated at:            2021-02-25T15:46:30Z   
Last Operation:                        
                       Status    create succeeded      
                       Message   Task not found
```

Get the GUID of the Secrets Manager instance,

```bash
SM_GUID=$(ibmcloud resource service-instance $SM_NAME --output json | jq -r '.[0].guid')
echo $SM_GUID
```

Construct the Secrets Manager API URL,

```bash
SM_API_URL=https://$SM_GUID.$REGION.secrets-manager.appdomain.cloud
echo $SM_API_URL
```

## Create an IBM Cloud API Key

```bash
IAM_APIKEY_NAME=$SM_NAME-apikey1
ibmcloud iam api-key-create $IAM_APIKEY_NAME -d "API key for Secrets Manager" --file $IAM_APIKEY_NAME.txt
IAM_APIKEY=$(cat $IAM_APIKEY_NAME.txt | jq -r '.apikey')
echo $IAM_APIKEY
```

## Create an Access Token

Using the `IAM_APIKEY` value for the IBM Cloud API Key, create and retrieve an access token,

```bash
IAM_TOKEN=$(curl -X POST \
  "https://iam.cloud.ibm.com/identity/token" \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'Accept: application/json' \
  --data-urlencode 'grant_type=urn:ibm:params:oauth:grant-type:apikey' \
  --data-urlencode apikey=$IAM_APIKEY | jq -r '.access_token')
```

Check if the access token was retrieved successfully,

```bash
$ echo $IAM_TOKEN

{"access_token":"eyJraWQiOiIyMDIxMDIxOTE4MzUiLCJhbGciOiJSUzI1NiJ9.eyJpYW1faWQiOiJJQk1pZC0zMTAwMDBGQ1A1IiwiaWQiOiJJQk1pZC0zMTAwMDBGQ1A1IiwicmVhbG1pZCI6IklCTWlkIiwianRpIjoiM2UzYjk5ODItN2Q5NC00NDVlLTk5OWQtMzgyOTQ4ZjE2ZmI4IiwiaWRlbnRpZmllciI6IjMxMDAwMEZDUDUiLCJnaXZlbl9uYW1lIjoiUkVNS08iLCJmYW1pbHlfbmFtZSI6IkRFIEtOSUtLRVIiLCJuYW1lIjoiUkVNS08gREUgS05JS0tFUiIsImVtYWlsIjoicmVta29oZGV2QHVzLmlibS5jb20iLCJzdWIiOiJyZW1rb2hkZXZAdXMuaWJtLmNvbSIsImF1dGhuIjp7InN1YiI6InJlbWtvaGRldkB1cy5pYm0uY29tIiwiaWFtX2lkIjoiaWFtLTMxMDAwMEZDUDUiLCJuYW1lIjoiUkVNS08gREUgS05JS0tFUiIsImdpdmVuX25hbWUiOiJSRU1LTyIsImZhbWlseV9uYW1lIjoiREUgS05JS0tFUiIsImVtYWlsIjoicmVta29oZGV2QHVzLmlibS5jb20ifSwiYWNjb3VudCI6eyJ2YWxpZCI6dHJ1ZSwiYnNzIjoiM2ZlM2MwZGUxOTcyNTdlZjYyZDgxYzlmOWMwZjMzYWEiLCJpbXNfdXNlcl9pZCI6IjYzOTE4NjMiLCJmcm96ZW4iOnRydWUsImltcyI6IjEwNTc2MDcifSwiaWF0IjoxNjE0MjY5NzU5LCJleHAiOjE2MTQyNzMzNTksImlzcyI6Imh0dHBzOi8vaWFtLmNsb3VkLmlibS5jb20vaWRlbnRpdHkiLCJncmFudF90eXBlIjoidXJuOmlibTpwYXJhbXM6b2F1dGg6Z3JhbnQtdHlwZTphcGlrZXkiLCJzY29wZSI6ImlibSBvcGVuaWQiLCJjbGllbnRfaWQiOiJkZWZhdWx0IiwiYWNyIjoxLCJhbXIiOlsicHdkIl19.Mx7jUxjnO8XFW0ZrnByYiUeU6dI7PnbRDEBRObx7H8RUMjkk4lQUbYxjGTgg6QficBKBjNVIhJ4dtzupqtkMy_cyXmgjMiUEmEAV4gTEaljcmFvJb27EvHhWUAv7uSOOQRLL_kByj_EcjohRB6Jwxdx3XNZbVkGgZ8sSnclrtWLGG35KKP5xghBUNYChfe-lbnQh_i5qszGsjJAa1uQ5zqjFRyRKwDkTR-sfOhBRh2legJOCeWsT5kxbtNw4xebOAJQHUQB087QElYJfMOkPpOyX7KMVvuctKGR--z_Rb5_lUyt_JSZTFDFR4SVfHrHA-5fhXv0Ykqv5HBRrd4yTgw","refresh_token":"not_supported","ims_user_id":6391863,"token_type":"Bearer","expires_in":3600,"expiration":1614273359,"scope":"ibm openid"}
```

## Using the Vault CLI

If you installed the `vault cli` you can alternatively use the Vault CLI instead of the API.

```bash
export VAULT_ADDR=$SM_API_URL

vault write -format=json auth/ibmcloud/login token=$IAM_TOKEN
vault secrets enable -path=secret/myapp/config kv
vault kv put secret/myapp/config username=$MY_USERNAME password=$MY_PASSWORD
```

For more details, see the documentation for [Vault Commands (CLI)](https://www.vaultproject.io/docs/commands).

## Create a Secret Group

Secrets are organized in secret groups.

Define the following bash environment variables,

```bash
MY_APP=guestbook
SECRET_GROUP=$MY_APP-secretgroup1
SECRET_GROUP_DESCRIPTION="secret group 1 for application $MY_APP"
echo $SECRET_GROUP_DESCRIPTION
```

Create a secret group,

```bash
$ curl -X POST "$SM_API_URL/api/v1/secret_groups" -H "Authorization: Bearer $IAM_TOKEN" -H "Accept: application/json" -H "Content-Type: application/json" -d '{"metadata": {"collection_type": "application/vnd.ibm.secrets-manager.secret.group+json", "collection_total": 1 }, "resources": [{ "name": "'"$SECRET_GROUP"'", "description": "'"$SECRET_GROUP_DESCRIPTION"'" }]}'

{"metadata":{"collection_type":"application/vnd.ibm.secrets-manager.secret.group+json","collection_total":1},"resources":[{"creation_date":"2021-03-04T19:18:46Z","description":"secret group 1 for applicati
on guestbook","id":"3a225644-8083-1ae6-2ef5-d3972c046899","last_update_date":"2021-03-04T19:18:46Z","name":"guestbook-secretgroup1","type":"application/vnd.ibm.secrets-manager.secret.group+json"}]}

$ SECRET_GROUP_ID=$( curl -X GET "$SM_API_URL/api/v1/secret_groups" -H "Authorization: Bearer $IAM_TOKEN" -H "Accept: application/json" | jq -r '.resources[0].id' )
$ echo $SECRET_GROUP_ID

3a225644-8083-1ae6-2ef5-d3972c046899
```

## Static versus Dynamic Secrets

In this tutorial, I use static secrets using a key-value pair. [Dynamic secrets](https://learn.hashicorp.com/tutorials/vault/database-secrets) can be generated for some systems on-demand by Vault. For when an app wants to access S3 storage like IBM Cloud Object Storage (COS), it asks Vault for credentials. Vault then generates COS credentials granting permission to accss the S3 bucket. If you want to use a dynamic secret to access a Cloud Object Storage instead see this tutorial [here](https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-tutorial-access-storage-bucket).

## Create a Static Secret

Use the following environment variables to create a secret,

```bash
USERNAME=remkohdev
PASSWORD=Passw0rd1
SECRET_NAME="mongodb-dev-creds-secret1"
SECRET_DESCRIPTION="username password credentials for Mongodb instance in dev environment"
EXP_DATE="2021-12-31T00:00:00Z"
DEPLOYMENT_ENV=dev
DEPLOYMENT_REGION=us-south
```

Create a static secret,

```bash
curl -X POST "$SM_API_URL/api/v1/secrets/username_password" -H "Authorization: Bearer $IAM_TOKEN" -H "Accept: application/json" -H "Content-Type: application/json" -d '{"metadata": { "collection_type": "application/vnd.ibm.secrets-manager.secret+json", "collection_total": 1 }, "resources": [{ "name": "'"$SECRET_NAME"'", "description": "'"$SECRET_DESCRIPTION"'", "secret_group_id": "'"$SECRET_GROUP_ID"'", "username": "'"$USERNAME"'", "password": "'"$PASSWORD"'", "expiration_date": "'"$EXP_DATE"'", "labels": [ "'"$DEPLOYMENT_ENV"'", "'"$DEPLOYMENT_REGION"'" ]}]}'
```

## Read Secret

Read a username_password type secret using the API,

```bash
$ curl -X GET "$SM_API_URL/api/v1/secrets" -H "Authorization: Bearer $IAM_TOKEN" -H "Accept: application/json"

{
    "metadata": {
        "collection_type": "application/vnd.ibm.secrets-manager.secret+json",
        "collection_total": 1
    },
    "resources": [
        {
            "created_by": "IBMid-310000FCP5",
            "creation_date": "2021-03-04T19:43:00Z",
            "crn": "crn:v1:bluemix:public:secrets-manager:us-south:a/3fe3c0de197257ef62d81c9f9c0f33aa:c84703a7-c37e-490f-ab8a-e24ddbcc0bcc:secret:5823ca61-20b9-75f4-804d-0d85a9fc58be",
            "description": "username password credentials for Mongodb instance in dev environment",
            "expiration_date": "2021-12-31T00:00:00Z",
            "id": "5823ca61-20b9-75f4-804d-0d85a9fc58be",
            "labels": [
                "dev",
                "us-south"
            ],
            "last_update_date": "2021-03-04T19:43:00Z",
            "name": "mongodb-dev-creds-secret1",
            "secret_group_id": "3a225644-8083-1ae6-2ef5-d3972c046899",
            "secret_type": "username_password",
            "state": 1,
            "state_description": "Active"
        }
    ]
}
```

## Conclusion

You're awesome! You stored database credentials in an encrypted secrets engine using Vault. A next step could be to inject the credentials into a pod running on Kubernetes or OpenShift to securely connect to a MongoDB database. Or you can explore using dynamic secrets with an S3 object storage like IBM Cloud Object Storage.
