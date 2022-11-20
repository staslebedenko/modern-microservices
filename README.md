# modern-microservices
modern microservices workshop

## Steps
The workshop is build around many steps.

0. Deployment of required Azure infrastructure, databases for step one and two, along with Container App environment and AKS
1. Preparing monolith for split and splitting it into two services with containerization.
2. Overview of the existing Azure resources and deployment of solution from Visual Studio
3. Overview of Container Apps key features and problems. Extra tasks
4. Adding DAPR to the Container App and explanation of approaches and benefits
5. Create AKS manifest, setting up DAPR in Azure Kubernetes cluster and deploying solution to Cloud.
6. Adding DAPR pubsub component and RabbitMQ container. Changing solution code to work with a pubsub.

## Prerequisites

1. Visual Studio or Visual Studio Code with .NET Framework 6.
2. Docker Desktop to run the containerized application locally.
https://www.docker.com/products/docker-desktop
3. DAPR CLI installed on a local machine.
https://docs.dapr.io/getting-started/install-dapr-cli/
4. AZ CLI tools installation(for cloud deployment)
https://aka.ms/installazurecliwindows
5. Azure subscription, if you want to deploy applications to Kubernetes(AKS).
https://azure.microsoft.com/en-us/free/
6. Kubectl installation https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows
7. Good mood :)

(optional)Kompose tool for Kubernetes manifest generation.
https://kompose.io/getting-started/ 

## Step 0. Azure infrastructure
Script below should be run via Azure Portal bash console. 
You will receive database connection strings with setx command as output of this script. Along with Application Insights key
Please add a correct name of your subscription to the first row of the script. 

As result of this deployemt you should open the local command line as admin and execute output strings from the script execution to set environment variables.
It is also good to store them in the text file for the future usage

You might need to reboot your PC so secrets will be available from the OS.

For the start the preferrable way is to use Azue CLI bash console via Azure portal.

```bash
subscriptionID=$(az account list --query "[?contains(name,'Microsoft')].[id]" -o tsv)
echo "Test subscription ID is = " $subscriptionID
az account set --subscription $subscriptionID
az account show

location=northeurope
postfix=$RANDOM

#----------------------------------------------------------------------------------
# Database infrastructure
#----------------------------------------------------------------------------------

export dbResourceGroup=dcc-modern-data$postfix
export dbServername=dcc-modern-sql$postfix
export dbPoolname=dbpool
export dbAdminlogin=FancyUser3
export dbAdminpassword=Sup3rStr0ng52$postfix
export dbPaperName=paperorders
export dbDeliveryName=deliveries

az group create --name $dbResourceGroup --location $location

az sql server create --resource-group $dbResourceGroup --name $dbServername --location $location \
--admin-user $dbAdminlogin --admin-password $dbAdminpassword
	
az sql elastic-pool create --resource-group $dbResourceGroup --server $dbServername --name $dbPoolname \
--edition Standard --dtu 50 --zone-redundant false --db-dtu-max 50

az sql db create --resource-group $dbResourceGroup --server $dbServername --elastic-pool $dbPoolname \
--name $dbPaperName --catalog-collation SQL_Latin1_General_CP1_CI_AS
	
az sql db create --resource-group $dbResourceGroup --server $dbServername --elastic-pool $dbPoolname \
--name $dbDeliveryName --catalog-collation SQL_Latin1_General_CP1_CI_AS	

sqlClientType=ado.net

SqlPaperString=$(az sql db show-connection-string --name $dbPaperName --server $dbServername --client $sqlClientType --output tsv)
SqlPaperString=${SqlPaperString/Password=<password>;}
SqlPaperString=${SqlPaperString/<username>/$dbAdminlogin}

SqlDeliveryString=$(az sql db show-connection-string --name $dbDeliveryName --server $dbServername --client $sqlClientType --output tsv)
SqlDeliveryString=${SqlDeliveryString/Password=<password>;}
SqlDeliveryString=${SqlDeliveryString/<username>/$dbAdminlogin}

SqlPaperPassword=$dbAdminpassword

#----------------------------------------------------------------------------------
# AKS infrastructure
#----------------------------------------------------------------------------------

location=northeurope
groupName=dcc-modern-cluster$postfix
clusterName=dcc-modern-cluster$postfix
registryName=dccmodernregistry$postfix


az group create --name $groupName --location $location

az acr create --resource-group $groupName --name $registryName --sku Standard
az acr identity assign --identities [system] --name $registryName

az aks create --resource-group $groupName --name $clusterName --node-count 3 --generate-ssh-keys --network-plugin azure
az aks update --resource-group $groupName --name $clusterName --attach-acr $registryName

#----------------------------------------------------------------------------------
# Service bus queue
#----------------------------------------------------------------------------------

groupName=dcc-modern-extras$postfix
location=northeurope
az group create --name $groupName --location $location
namespaceName=dccModern$postfix
queueName=createdelivery

az servicebus namespace create --resource-group $groupName --name $namespaceName --location $location
az servicebus queue create --resource-group $groupName --name $queueName --namespace-name $namespaceName

serviceBusString=$(az servicebus namespace authorization-rule keys list --resource-group $groupName --namespace-name $namespaceName --name RootManageSharedAccessKey --query primaryConnectionString --output tsv)

#----------------------------------------------------------------------------------
# Application insights
#----------------------------------------------------------------------------------

insightsName=dccmodernlogs$postfix
az monitor app-insights component create --resource-group $groupName --app $insightsName --location $location --kind web --application-type web --retention-time 120

instrumentationKey=$(az monitor app-insights component show --resource-group $groupName --app $insightsName --query  "instrumentationKey" --output tsv)

#----------------------------------------------------------------------------------
# Azure Container Apps
#----------------------------------------------------------------------------------

az extension add --name containerapp --upgrade

az provider register --namespace Microsoft.App

az provider register --namespace Microsoft.OperationalInsights

acaGroupName=dcc-modern-containerapp$postfix
location=northeurope
logAnalyticsWorkspace=dcc-modern-logs$postfix
containerAppsEnv=dcc-environment$postfix

az group create --name $acaGroupName --location $location

az monitor log-analytics workspace create \
--resource-group $acaGroupName --workspace-name $logAnalyticsWorkspace

logAnalyticsWorkspaceClientId=`az monitor log-analytics workspace show --query customerId -g $acaGroupName -n $logAnalyticsWorkspace -o tsv | tr -d '[:space:]'`

logAnalyticsWorkspaceClientSecret=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g $acaGroupName -n $logAnalyticsWorkspace -o tsv | tr -d '[:space:]'`

az containerapp env create \
--name $containerAppsEnv \
--resource-group $acaGroupName \
--logs-workspace-id $logAnalyticsWorkspaceClientId \
--logs-workspace-key $logAnalyticsWorkspaceClientSecret \
--dapr-instrumentation-key $instrumentationKey \
--logs-destination log-analytics \
--location $location

az containerapp env show --resource-group $acaGroupName --name $containerAppsEnv

# we don't need a section below for this workshop, but you can use it later
# use command below to fill credentials values if you want to use section below 
#az acr credential show --name $registryName 

#imageName=<CONTAINER_IMAGE_NAME>
#acrServer=<REGISTRY_SERVER>
#acrUser=<REGISTRY_USERNAME>
#acrPassword=<REGISTRY_PASSWORD>

#az containerapp create \
#  --name my-container-app \
#  --resource-group $acaGroupName \
#  --image $imageName \
#  --environment $containerAppsEnv \
#  --registry-server $acrServer \
#  --registry-username $acrUser \
#  --registry-password $acrPassword

#----------------------------------------------------------------------------------
# Azure Key Vault with secrets assignment and access setup
#----------------------------------------------------------------------------------

keyvaultName=dcc-modern$postfix
principalName=vaultadmin
principalCertName=vaultadmincert

az keyvault create --resource-group $groupName --name $keyvaultName --location $location
az keyvault secret set --name SqlPaperPassword --vault-name $keyvaultName --value $SqlPaperPassword

az ad sp create-for-rbac --name $principalName --create-cert --cert $principalCertName --keyvault $keyvaultName --skip-assignment --years 3

# get appId from output of step above and add it after --id in command below.

# az ad sp show --id 474f817c-7eba-4656-ae09-979a4bc8d844
# get object Id (located before info object) from command output above and set it to command below 

# az keyvault set-policy --name $keyvaultName --object-id f1d1a707-1356-4fb8-841b-98e1d9557b05 --secret-permissions get
#----------------------------------------------------------------------------------
# SQL connection strings
#----------------------------------------------------------------------------------

printf "\n\nRun string below in local cmd prompt to assign secret to environment variable SqlPaperString:\nsetx SqlPaperString \"$SqlPaperString\"\n\n"
printf "\n\nRun string below in local cmd prompt to assign secret to environment variable SqlDeliveryString:\nsetx SqlDeliveryString \"$SqlDeliveryString\"\n\n"
printf "\n\nRun string below in local cmd prompt to assign secret to environment variable SqlPaperPassword:\nsetx SqlPaperPassword \"$SqlPaperPassword\"\n\n"
printf "\n\nRun string below in local cmd prompt to assign secret to environment variable SqlDeliveryPassword:\nsetx SqlDeliveryPassword \"$SqlPaperPassword\"\n\n"
printf "\n\nRun string below in local cmd prompt to assign secret to environment variable ServiceBusString:\nsetx ServiceBusString \"$serviceBusString\"\n\n"

echo "Update open-telemetry-collector-appinsights.yaml in Step 5 End => <INSTRUMENTATION-KEY> value with:  " $instrumentationKey
```

## Step 1. Monolith split and containerization
The Start repository contains approach for splitting monolithic app into parts inside the same solution and single database with two schemas

The End folder contains solution with the one solution and two projects.

First we adding docker containerization via context menu of each project.

Then we adding orchestration support via docker compose again to the each project

Following by adding the environment variable file to the root folder

And adding to the Order controller the method to call the Delivery container endpoint

We will also configure the Compose startup via Visual Studio so the Paper project endpoint will start as a primary starting point.

All changes available in the commit history, so it is easy to track them.

Don't forget to add env file with a secrets content along with changes to docker-compose.yaml in the root folder.

Solution will work with two containers, so there is a need to put the correct container port for Delivery service.

!! Be aware, if you have docker build exceptions in Visual studio with errors related to the File system, there is a need to configure docker desktop. 
Open Docker desktop => configuration => Resources => File sharing => Add your project folder or entire drive, C:\ for example. Dont forget to remove drive setting later on.

When you try to start the same solution from the new folder, you need to stop and delete containers via docker compose.

## Step 2. Azure infrastructure, deployment and GitHub actions

We will start this step with the overview of the infrastructure deployed in Azure

Changes to the secrets to get a proper secrets for our application
Addition of application insights nuget and custom logging
Changes to URLs

Deployment to Container Apps via Visual studio
Configuration for GitHub actions deployments

![image](https://user-images.githubusercontent.com/36765741/202031699-2787a2b0-2368-45e5-b37c-76e1550b78d7.png)

![image](https://user-images.githubusercontent.com/36765741/202032521-3797b950-d41f-4b43-8055-70ca53c3fd70.png)

![image](https://user-images.githubusercontent.com/36765741/202032630-01472083-e44d-4026-8e23-1bf2f82b6dfd.png)

![image](https://user-images.githubusercontent.com/36765741/202032687-b1cda810-8675-4120-9b80-28c0fc943087.png)

![image](https://user-images.githubusercontent.com/36765741/202032789-d6b845a6-9608-4b66-878b-6bea22fab75e.png)

![image](https://user-images.githubusercontent.com/36765741/202032840-2d215c00-cb11-402a-8983-56bffb511add.png)

But if we will try to publish our first app, it will fail, because of the bad docker file generated by Visual studio.

Let us fix it by adding string below, so dockew would know which step it should take
```
build AS publish
```
before the following line
```
RUN dotnet publish "TPaperOrders.csproj" -c Release -o /app/publish /p:UseAppHost=false
```

and also adding the port 443 for SSL, please repeat this step for delivery project if you got the bad manifest

Let's try to deploy two containers to container app.

And now it is time to figure out our FQDN so we can call our services externally and internally.
And also have a look into our Portal configurations

And after a brief look with can figure out that deployment has failed
![image](https://user-images.githubusercontent.com/36765741/202293909-ab636822-5c15-4537-b58d-18c35957e911.png)

after a click on the failed deployment, we need to select Console logs and see results in the log analytics

After expanding the latest log entry, we can see Unhandled exception. System.ArgumentNullException: Value cannot be null. (Parameter 'Password')
Which means that we not configured secrets for our application.

Let's do a quick fix by adding secrets to the Container App settings (KeyVault we will use later)
![image](https://user-images.githubusercontent.com/36765741/202296088-7dc6f0cb-538a-4e2a-a459-5151c2038a01.png)

After this changes we re-deploying our application and getting another error Cannot open server 'dcc-modern-sql' requested by the login. Client with IP address

There is a two ways to solve this problem, the first is to use Azure Connector preview and make a direct link to database with secrets managed by KeyVault, or add IP address to exceptions. Or you can create a Container app environment with VNet from the start and use network endpoint of Azure SQL

After this changes our application successfuly provisioned.

There is a limit of one ingress per one container app, along with the reccomendation to host two containers only in case of workload + sidecar.
Moreover Visual studion will not let you easily do so.

So we need to deploy a delivery app as a separate app and configure urls for service to service communications.

We will need to get a full URL of Delivery Container APP from a portal or Azure cli for automation
```
acaGroupName=dcc-modern-containerapp
acaName=dcc-modern-containerapp
								  
az containerapp show --resource-group acaGroupName \
--name acaName --query properties.configuration.ingress.fqdn
```
so we will get following url

```
tpaperdelivery-app-2022111622390--dgcp1or.agreeablecoast-99a44d4d.northeurope.azurecontainerapps.io
```
and add internal, so it will look like
```
tpaperdelivery-app-2022111622390--dgcp1or.internal.agreeablecoast-99a44d4d.northeurope.azurecontainerapps.io
```
and add this value to DeliveryUrl environment variable file with docker url, and to container app config DeliveryUrl

And please allow Insecure connections for Delivery service via ingress configuration
![image](https://user-images.githubusercontent.com/36765741/202306805-5620b8cd-4fb1-4dba-80b7-8ce54e80dbc9.png)

We should use the full path to make initial call to order API and see results
tpaperorders-app-20221115224238--k1osno8.agreeablecoast-99a44d4d.northeurope.azurecontainerapps.io/api/order/create/1

And now it is time to add intialize and add DAPR, run solution locally, initialize DAPR in the ACA apps and add pubsub component via Azure Service Bus.

Lets open a command prompt with admin rights to install Dapr CLI
```
powershell -Command "iwr -useb https://raw.githubusercontent.com/dapr/cli/master/install/install.ps1 | iex"
```

And initialize DAPR for the local development

```
dapr init
```
We can check results(container list) by opening Docker Desktop or running command docker ps.

Lets update our solution compose file with DAPR sidecar containers, for production you should replace DAPR latest with particular version to avoid problems with auto updates.

```
version: '3.4'

services:
  tpaperdelivery:
    image: ${DOCKER_REGISTRY-}tpaperdelivery
    build:
      context: .
      dockerfile: TPaperDelivery/Dockerfile
    ports:
      - "52000:50001"
    env_file:
      - settings.env
      
  tpaperdelivery-dapr:
    image: "daprio/daprd:latest"
    command: [ "./daprd", "-app-id", "tpaperdelivery", "-app-port", "80" ]
    depends_on:
      - tpaperdelivery
    network_mode: "service:tpaperdelivery"
      
  tpaperorders:
    image: ${DOCKER_REGISTRY-}tpaperorders
    build:
      context: .
      dockerfile: TPaperOrders/Dockerfile
    ports:
      - "51000:50001"
    env_file:
      - settings.env

  tpaperorders-dapr:
    image: "daprio/daprd:latest"
    command: [ "./daprd", "-app-id", "tpaperorders", "-app-port", "80" ]
    depends_on:
      - tpaperorders
    network_mode: "service:tpaperorders"      
```

Now we need to make adjustments to PaperOrders project, by adding DAPR dependency

```
<PackageReference Include="Dapr.AspNetCore" Version="1.9.0" />
```
Adding it to the Startup 

```
services.AddControllers().AddDapr();
```
And replacing method CreateDeliveryForOrder with http endpoint invocation via DAPR client

```
        private async Task<DeliveryModel> CreateDeliveryForOrder(EdiOrder savedOrder, CancellationToken cts)
        {
            string serviceName = "tpaperdelivery";
            string route = $"api/delivery/create/{savedOrder.ClientId}/{savedOrder.Id}/{savedOrder.ProductCode}/{savedOrder.Quantity}";

            DeliveryModel savedDelivery = await _daprClient.InvokeMethodAsync<DeliveryModel>(
                                          HttpMethod.Get, serviceName, route, cts);

            return savedDelivery;
        }
        }
```

If we will start a service and invoke a new order via http://localhost:52043/api/order/create/1 we can see that everything working as usual, except that we got additional container sidecars for each service 

This way we leveraged service locator provided by dapr, it is still a http communication between services, but now you can skip usage of evnironment variable and routing will be a responsibility of DAPR.

### Now we will need to add a PubSub component and DAPR component 

First we preparing simplified DAPR pubsub yaml manifest for pubsub(available in yaml folder of Step 2 End)

```
componentType: pubsub.azure.servicebus
version: v1
metadata:
- name: connectionString
  secretRef: sbus-connectionstring
secrets:
- name: sbus-connectionstring
  value: super-secret
```

And then deploying it to Azure via local azure CLI or portal console with file upload. Locally you should do az login first. 
The pubsub component would be pubsubsbus, we will use it later in code.
```
az containerapp env dapr-component set --resource-group dcc-modern-containerapp --name dcc-environment --dapr-component-name pubsubsbus --yaml "pubsubsbus.yaml"
```

Important thing, we are adding this DAPR component for entire environment, so it will be available for all apps, also we not included scopes at this step, something for the future.

Afterwards there is a need to put the correct Azure Service Bus connection string on the portal via Container App environment
![image](https://user-images.githubusercontent.com/36765741/202546502-342421d4-c14f-4f2f-8118-df220f752232.png)

One important this, please add the following section to your service bus connection string ";EntityPath=createdelivery"
"Endpoint=sb://dccmodern2141.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=CGnGz1L+Jw=;EntityPath=createdelivery"

Now let's add DAPR pub/sub components to our solution.

Change CreateDeliveryForOrder in TPaperOrders project to the following code that uses DAPR pubsub
```
        private async Task<DeliveryModel> CreateDeliveryForOrder(EdiOrder savedOrder, CancellationToken cts)
        {
            var newDelivery = new DeliveryModel
            {
                Id = 0,
                ClientId = savedOrder.ClientId,
                EdiOrderId = savedOrder.Id,
                Number = savedOrder.Quantity,
                ProductId = 0,
                ProductCode = savedOrder.ProductCode,
                Notes = "Prepared for shipment"
            };

            await _daprClient.PublishEventAsync<DeliveryModel>("pubsub", "createdelivery", newDelivery, cts);

            return newDelivery;
        }
```

Adding DAPR dependency
```
 <PackageReference Include="Dapr.AspNetCore" Version="1.9.0" />
```

Change method CreateDeliveryForOrder in OrderController to
```
        private async Task<DeliveryModel> CreateDeliveryForOrder(EdiOrder savedOrder, CancellationToken cts)
        {
            var newDelivery = new DeliveryModel
            {
                Id = 0,
                ClientId = savedOrder.ClientId,
                EdiOrderId = savedOrder.Id,
                Number = savedOrder.Quantity,
                ProductId = 0,
                ProductCode = savedOrder.ProductCode,
                Notes = "Prepared for shipment"
            };

            await _daprClient.PublishEventAsync<DeliveryModel>("pubsubsbus", "createdelivery", newDelivery, cts);

            return newDelivery;
        }
```

And most importantly change double to decimal in Delivery model Number field


For TPaperDelivery project, update Startup class
```
services.AddControllers().AddDapr();
```
and 
```
app.UseAuthorization();

app.UseCloudEvents();

app.UseOpenApi();
app.UseSwaggerUi3();

app.UseEndpoints(endpoints =>
{
endpoints.MapSubscribeHandler();
endpoints.MapControllers();
});
```

And finally make a proper signature on ProcessEdiOrder endpoint
```
[Topic("pubsubsbus", "createdelivery")]
[HttpPost]
[Route("createdelivery")]
```

And add method to enumerate all stored deliveries
```
[HttpGet]
[Route("deliveries")]
public async Task<IActionResult> Get(CancellationToken cts)
{
    Delivery[] registeredDeliveries = await _context.Delivery.ToArrayAsync(cts);

    return new OkObjectResult(registeredDeliveries);
}
```

And after all this changes we can deploye and observe results in Azure.


## Step 3. Overview of Container Apps key features and problems

Additional tasks

* Return delivery model via additional pubsub to Order service, so we can update order entity with a proper delivery Id.
* Add a local pubsub via Rabbit MQ for debug purposes(optional)
* Add deployment via GitHub actions(optional)


## Step 4. Migrating solution to Kubernetes and switching pubsub to RabbitMQ

As our application grows, Container Apps might be not enough, so the essential migration path is to Azure Kubernetes Service

Spoiler. We will add Azure KeyVault in the next step and continue with abstracting it with DAPR component

First we need to add two secrets to the K8s manifest
Service bus connection string and SQL server password

We will need to encode secrets with base64 via command line or online tool https://string-functions.com/base64encode.aspx

You can check example conversion below
```
Endpoint=sb://dccmodern3214.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=J+Jw=;EntityPath=createdelivery
RW5kcG9pbnQ9c2I6Ly9kY2Ntb2Rlcm4zMjE0LnNlcnZpY2VidXMud2luZG93cy5uZXQvO1NoYXJlZEFjY2Vzc0tleU5hbWU9Um9vdE1hbmFnZVNoYXJlZEFjY2Vzc0tleTtTaGFyZWRBY2Nlc3NLZXk9SitKdz07RW50aXR5UGF0aD1jcmVhdGVkZWxpdmVyeQ==
```

And so our final manifest will look like this.
```
apiVersion: v1
kind: Secret
metadata:
  name: sbus-secret
type: Opaque
data:
  connectionstring: RW5kcG9mVyeQ==
---
apiVersion: v1
kind: Secret
metadata:
  name: sql-secret
type: Opaque
data:
  password: U3Vg==
```

Lets start with CMD. !!!Use additional command az account set --subscription 95cd9078f8c to deploy resources into the correct subscription
```cmd
az login

az account show
az acr login --name dccmodernregistry
az aks get-credentials --resource-group dcc-modern-cluster --name dcc-modern-cluster
kubectl config use-context dcc-modern-cluster
kubectl get all
```

And initialize DAPR in our kubernetes cluster
```cmd
dapr init -k 
```

Validate results of initialization
```cmd
dapr status -k 
```
Then we will need to build our solution in release mode and observe results with command. We building containers trough Visual Studio and tagging them via command line
You can also start docker desktop application for GUI container handling.

Lets see what images do we have with
```cmd
docker images
```

Lets tag our newly built container with azure container registry name and version.
```cmd
docker tag tpaperorders:latest dccmodernregistry.azurecr.io/tpaperorders:v1
docker tag tpaperdelivery:latest dccmodernregistry.azurecr.io/tpaperdelivery:v1
```

Check results with
```cmd
docker images
```


And push images to container registry
```cmd
docker push dccmodernregistry.azurecr.io/tpaperorders:v1
docker push dccmodernregistry.azurecr.io/tpaperdelivery:v1
```

Now we need to update container version in orders manifest
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tpaperorders
  namespace: tpaper
  labels:
    app: tpaperorders
spec:
  replicas: 1
  selector:
    matchLabels:
      service: tpaperorders
  template:
    metadata:
      labels:
        app: tpaperorders
        service: tpaperorders
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "tpaperorders"
        dapr.io/app-port: "80"
        dapr.io/log-level: debug
    spec:
      containers:
        - name: tpaperorders
          image: msactionregistry.azurecr.io/tpaperorders:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 80
              protocol: TCP
          env:
            - name: ASPNETCORE_URLS
              value: http://+:80
            - name: SqlPaperString
              value: Server=tcp:dcc-modern-sql.database.windows.net,1433;Database=paperorders;User ID=FancyUser3;Encrypt=true;Connection Timeout=30;
            - name: SqlPaperPassword
              valueFrom:
                secretKeyRef:
                  name: sql-secret
                  key: password
---
apiVersion: v1
kind: Service
metadata:
  name: tpaperorders
  namespace: tpaper
  labels:
    app: tpaperorders
    service: tpaperorders
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    service: tpaperorders
```

And delivery manifest
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tpaperdelivery
  namespace: tpaper
  labels:
    app: tpaperdelivery
spec:
  replicas: 1
  selector:
    matchLabels:
      service: tpaperdelivery
  template:
    metadata:
      labels:
        app: tpaperdelivery
        service: tpaperdelivery
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "tpaperdelivery"
        dapr.io/app-port: "80"
        dapr.io/log-level: debug
    spec:
      containers:
        - name: tpaperdelivery
          image: dccmodernregistry.azurecr.io/tpaperdelivery:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 80
              protocol: TCP
          env:
            - name: ASPNETCORE_URLS
              value: http://+:80
            - name: SqlDeliveryString
              value: Server=tcp:dcc-modern-sql.database.windows.net,1433;Database=deliveries;User ID=FancyUser3;Encrypt=true;Connection Timeout=30;
            - name: SqlPaperPassword
              valueFrom:
                secretKeyRef:
                  name: sql-secret
                  key: password
---
apiVersion: v1
kind: Service
metadata:
  name: tpaperdelivery
  namespace: tpaper
  labels:
    app: tpaperdelivery
    service: tpaperdelivery
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    service: tpaperdelivery
```

But before deployment of our services we need to deploye secrets and pubsub DAPR component
As you can see, the file is slightly different from Container Apps version

DAPR pubsub component
```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub-super-new
  namespace: default
spec:
  type: pubsub.azure.servicebus
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: sbus-secret
      key:  connectionstring
auth:
  secretStore: kubernetes
scopes:
  - tpaperorders
  - tpaperdeliver
```

Manifest to create a new namespace
```
apiVersion: v1
kind: Namespace
metadata:
  name: tpaper
  labels:
    name: tpaper
```

deployment of secrets and pubsub component
```cmd
kubectl apply -f aks_secrets.yaml
kubectl apply -f aks_pubsub-servicebus.yaml
```

creation of the new namespace, so we can easily find our services
```cmd
kubectl apply -f aks_namespace-tpaper.yaml
```

Now we need to pray the "demo gods" for our deployment and run commands below
```cmd
kubectl apply -f aks_tpaperorders-deploy.yaml
kubectl apply -f aks_tpaperdelivery-deploy.yaml
```


You can use set of commands below for quick container/publish re-deployments.
Just change version in kubernetes manifest and commands below.
```cmd
docker tag tpaperorders:latest dccmodernregistry.azurecr.io/tpaperorders:v1
docker images
docker push dccmodernregistry.azurecr.io/tpaperorders:v1
kubectl apply -f aks_tpaperorders-deploy.yaml
kubectl get all --all-namespaces

docker tag tpaperdelivery:latest dccmodernregistry.azurecr.io/tpaperdelivery:v1
docker images
docker push dccmodernregistry.azurecr.io/tpaperdelivery:v1
kubectl apply -f aks_tpaperorders-deploy.yaml
kubectl get all --all-namespaces
```

## Step 5. Additional DAPR components.


## Step 6.


## Step 7.
