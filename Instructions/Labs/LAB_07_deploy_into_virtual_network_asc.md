---
lab:
    title: 'Lab: Protect endpoints using Web Application Firewalls'
    module: 'Module 7: Protect endpoints using Web Application Firewall'
---

# Challenge: Protect endpoints using Web Application Firewall
# Student manual

## Challenge scenario

By now, you have completed setting up your Spring Petclinic application in Azure and secured the secrets used by the microservices to connect to their data store. You are satisfied with the results, but you do recognize that there is still room for improvements. In particular, you are concerned with the public endpoints of the application which are directly accessible to anyone with access to the internet. You would like to add a Web Application Firewall to filter incoming requests to your application. In this exercise, you will step through implementing this configuration.

## Objectives

After you complete this challenge, you will be able to:

- Create networking resources
- Recreate Azure Spring Apps service and apps in the virtual network
- Configure a private DNS zone
- Acquire a certificate and add it to Key Vault
- Configure a domain in Azure Spring Apps
- Create the Application Gateway resources
- Access the application by DNS name
- Configure WAF on Application Gateway

## Challenge Duration

- **Estimated Time**: 60 minutes

## Instructions

During this challenge, you'll:

- Create networking resources
- Recreate Azure Spring Apps service and apps in the virtual network
- Configure a private DNS zone
- Acquire a certificate and add it to Key Vault
- Configure a domain in Azure Spring Apps
- Create the Application Gateway resources
- Access the application by DNS name
- Configure WAF on Application Gateway

   > **Note**: The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.

### Create networking resources

Since you want to place apps in your Azure Spring Cloud service behind an Azure Application Gateway, you will need to provide the networking resources for the Spring Cloud service and the Application Gateway. You can deploy all of them in the same virtual network, in which case you will need at least 3 subnets, with one of them for the Application Gateway and 2 for the Spring Cloud service. You will also need to create a subnet for private endpoints that provide connectivity to any backend services your applications use, such as the Azure Database for MySQL Single Server instance and the Azure Key Vault instance. You can use the following guidance to implement these changes.

[Create a Virtual Network and default subnet](https://docs.microsoft.com/en-us/cli/azure/network/vnet?view=azure-cli-latest#az-network-vnet-create)
[Add subnets to a Virtual Network](https://docs.microsoft.com/en-us/cli/azure/network/vnet/subnet?view=azure-cli-latest)
[Deploy Azure Spring Cloud in a virtual network](https://docs.microsoft.com/en-us/azure/spring-cloud/how-to-deploy-in-azure-virtual-network?tabs=azure-portal)

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash prompt, run the following command to create a virtual network.

   ```bash
   VIRTUAL_NETWORK_NAME=springcloudvnet
   az network vnet create --resource-group $RESOURCE_GROUP \
       --name $VIRTUAL_NETWORK_NAME \
       --location $LOCATION \
       --address-prefix 10.1.0.0/16
   ```

1. Create 2 subnets intended for hosting Azure Spring Cloud in this virtual network, 1 subnet intended for Application Gateway and 1 subnet for the private endpoints of the Azure Database for MySQL Single Server instance and the Azure Key Vault instance. Store the subnet names in environment variables, which will allow you to reference them later in this exercise:

   ```bash
   SERVICE_RUNTIME_SUBNET_CIDR=10.1.0.0/24
   APP_SUBNET_CIDR=10.1.1.0/24
   APPLICATION_GATEWAY_SUBNET_CIDR=10.1.2.0/24
   PRIVATE_ENDPOINTS_SUBNET_CIDR=10.1.3.0/24
   APPLICATION_GATEWAY_SUBNET_NAME=app-gw-subnet
   PRIVATE_ENDPOINTS_SUBNET_NAME=private-endpoints-subnet
   az network vnet subnet create --resource-group $RESOURCE_GROUP \
       --vnet-name $VIRTUAL_NETWORK_NAME \
       --address-prefixes $SERVICE_RUNTIME_SUBNET_CIDR \
       --name service-runtime-subnet 
   az network vnet subnet create --resource-group $RESOURCE_GROUP \
       --vnet-name $VIRTUAL_NETWORK_NAME \
       --address-prefixes $APP_SUBNET_CIDR \
       --name apps-subnet
   az network vnet subnet create \
       --name $APPLICATION_GATEWAY_SUBNET_NAME \
       --resource-group $RESOURCE_GROUP \
       --vnet-name $VIRTUAL_NETWORK_NAME \
       --address-prefix $APPLICATION_GATEWAY_SUBNET_CIDR
   az network vnet subnet create \
       --name $PRIVATE_ENDPOINTS_SUBNET_NAME \
       --resource-group $RESOURCE_GROUP \
       --vnet-name $VIRTUAL_NETWORK_NAME \
       --address-prefix $PRIVATE_ENDPOINTS_SUBNET_CIDR
   ```

1. Assign the Owner role-based access control (RBAC) role to the Azure Service Provider for Spring Cloud access in the scope of the newly created virtual network. This will allow the resource provider to create its resources in the service-runtime-subnet and apps-subnet subnets. The GUID used in the second command is the service provider id for Azure Spring Cloud.

   > **Note**: The `export MSYS_NO_PATHCONV=1` must be included to address an issue with implementing role assignment when using Azure CLI in Git Bash shell, as documented on [GitHub](https://github.com/Azure/azure-cli/issues/16317).

   ```bash
   VIRTUAL_NETWORK_RESOURCE_ID=`az network vnet show \
       --name $VIRTUAL_NETWORK_NAME \
       --resource-group $RESOURCE_GROUP \
       --query "id" \
       --output tsv`

   export MSYS_NO_PATHCONV=1

   az role assignment create \
       --role "Owner" \
       --scope $VIRTUAL_NETWORK_RESOURCE_ID \
       --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2
   ```

</details>

### Recreate Azure Spring Cloud service and apps in the virtual network

Now that you have all the networking resources ready, you need to recreate your Azure Spring Cloud service within this virtual network. Before you do so, delete your existing Azure Spring Cloud instance first. You can use the following guidance to perform this task.

[Deploy Azure Spring Cloud in a virtual network](https://docs.microsoft.com/en-us/azure/spring-cloud/how-to-deploy-in-azure-virtual-network?tabs=azure-CLI)

When you recreate your Spring Cloud instance in the virtual network, you will also need to rerun some of the steps from the previous exercise: 
- recreate the config server.
- recreate and redeploy all apps. In the previous exercises you assigned an endpoint to the api-gateway and admin service apps. At this point, in this task, you will skip this step and, instead, you will do so later in this exercise, once you configure the internal DNS name resolution.
- reassign a managed identity to each of the apps and give them access to the Azure Key Vault so they can access the username and password secrets required to connect to the MySQL database.

<details>
<summary>hint</summary>
<br/>

1. To start, delete your existing Azure Spring Cloud instance by running the following command from the Git Bash shell prompt.

   ```bash
   az spring-cloud delete \
       --name $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP
   ```

1. Next, recreate your Azure Spring Cloud instance within the designated subnets of the virtual network you created earlier in this exercise.

   ```bash
   SPRING_CLOUD_SERVICE=springcloudsvc$RANDOM$RANDOM
   az config set defaults.group=$RESOURCE_GROUP defaults.spring-cloud=$SPRING_CLOUD_SERVICE
   az spring-cloud create  \
       --resource-group $RESOURCE_GROUP \
       --name $SPRING_CLOUD_SERVICE \
       --vnet $VIRTUAL_NETWORK_NAME \
       --service-runtime-subnet service-runtime-subnet \
       --app-subnet apps-subnet \
       --sku standard \
       --location $LOCATION
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 15 minutes.

1. Set up the config server.

   ```bash
   az spring-cloud config-server git set --name $SPRING_CLOUD_SERVICE \
                                         --resource-group $RESOURCE_GROUP \
                                         --uri $GIT_REPO \
                                         --label main \
                                         --password $GIT_PASSWORD \
                                         --username $GIT_USERNAME
   ```

1. Recreate each of the apps in Spring Cloud, including managed identities for the customers-service, visits-service, and vets-service apps.

   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name api-gateway

   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name admin-service
                        
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name customers-service \
                              --system-assigned

   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name visits-service \
                              --system-assigned

   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name vets-service \
                              --system-assigned
   ```

1. Retrieve the managed identities of the customers, visits and vets apps, and grant them access to the Key Vault instance.

   ```bash
   CUSTOMERS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name customers-service \
       --output tsv \
       --query principalId)

   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $CUSTOMERS_SERVICE_ID

   VISITS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name visits-service \
       --output tsv \
       --query principalId)

   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VISITS_SERVICE_ID

   VNETS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name vets-service \
       --output tsv \
       --query principalId)

   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VNETS_SERVICE_ID
   ```

1. Redeploy each of the apps.

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name api-gateway \
                              --no-wait \
                              --artifact-path spring-petclinic-api-gateway/target/spring-petclinic-api-gateway-2.6.3.jar

   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name admin-service \
                              --no-wait \
                              --artifact-path spring-petclinic-admin-server/target/spring-petclinic-admin-server-2.6.3.jar
                        
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name customers-service \
                              --no-wait \
                              --artifact-path spring-petclinic-customers-service/target/spring-petclinic-customers-service-2.6.3.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql

   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name visits-service \
                              --no-wait \
                              --artifact-path spring-petclinic-visits-service/target/spring-petclinic-visits-service-2.6.3.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql

   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name vets-service \
                              --no-wait \
                              --artifact-path spring-petclinic-vets-service/target/spring-petclinic-vets-service-2.6.3.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql
   ```

</details>

### Configure a private DNS zone

At this point, you have redeployed your Azure Spring Cloud service in a virtual network, along with all of its apps. As the next step, to implement its connectivity without relying on a public endpoint, you need to set up a private DNS service for your apps so they are discoverable within the virtual network. You can use the following guidance to perform this task.

[Access your application in a private network](https://docs.microsoft.com/en-us/azure/spring-cloud/access-app-virtual-network?tabs=azure-CLI)

<details>
<summary>hint</summary>
<br/>

1. Start by identifying the IP address used by your Spring Cloud service. You can accomplish this by querying for the internal load balancer IP address of the service runtime network.

   ```bash
   SERVICE_RUNTIME_RG=`az spring-cloud show \
       --resource-group $RESOURCE_GROUP \
       --name $SPRING_CLOUD_SERVICE \
       --query "properties.networkProfile.serviceRuntimeNetworkResourceGroup" \
       --output tsv`
   IP_ADDRESS=`az network lb frontend-ip list \
       --lb-name kubernetes-internal \
       --resource-group $SERVICE_RUNTIME_RG \
       --query "[0].privateIpAddress" \
       --output tsv`
   ```

1. Next, create a private DNS zone to forward name resolution requests targeting the private.azuremicroservices.io namespace to this internal IP address.

   ```bash
   az network private-dns zone create \
       --resource-group $RESOURCE_GROUP \
       --name private.azuremicroservices.io
   ```

1. Link this private DNS zone to your virtual network.

   ```bash
   az network private-dns link vnet create \
       --resource-group $RESOURCE_GROUP \
       --name azure-spring-cloud-dns-link \
       --zone-name private.azuremicroservices.io \
       --virtual-network $VIRTUAL_NETWORK_NAME \
       --registration-enabled false
   ```

1. Now you need to create an A DNS record that will resolve the name associated with your Azure Spring Cloud service to the private IP address you identified earlier in this task.

   ```bash
   az network private-dns record-set a add-record \
       --resource-group $RESOURCE_GROUP \
       --zone-name private.azuremicroservices.io \
       --record-set-name '*' \
       --ipv4-address $IP_ADDRESS
   ```

1. Lastly you need to update your api-gateway and admin-service apps to retrieve the fully qualified domain name (FQDN) on your private DNS zone.

   ```bash
   az spring-cloud app update \
       --resource-group $RESOURCE_GROUP \
       --name api-gateway \
       --service $SPRING_CLOUD_SERVICE \
       --assign-endpoint true

   az spring-cloud app update \
       --resource-group $RESOURCE_GROUP \
       --name admin-service \
       --service $SPRING_CLOUD_SERVICE \
       --assign-endpoint true
   ```

   > **Note**: If you try connecting at this point to the spring petclinic application via the **api-gateway** and **admin-service** endpoints, you will not be able to do so, since these endpoints are currently only available within the virtual network. You could test such connectivity if you had an Azure VM connected to that virtual network. Later in this exercise, you will expose these two endpoints by using an Azure Application Gateway, which will allow you to test connectivity from the internet.

</details>

### Acquire a certificate and add it to Key Vault

You now have Spring Cloud service redeployed into a virtual network with a private DNS zone providing its name resolution. This configuration allows microservices to communicate with each other within the virtual network. However, to make the corresponding apps accessible from the internet, you need to implement a service that exposes a public endpoint. You will use for this purpose Azure Application Gateway. To accomplish this, you will also need to make sure that the domain name associated with the endpoint is the same as the name that Application Gateway uses to direct the traffic to the Azure Spring Cloud back end. This is required in order for cookies and generated redirect URLs to work as expected.

To configure this, you need to set up a custom domain name and generate a corresponding certificate for your Azure Spring Cloud apps. The certificate will be stored in the Azure Key Vault instance you created in the previous exercise and will be retrieved from there by your apps. In this exercise, for the simplicity sake, you will use a self-signed certificate. Keep in mind that, in production scenarios, you should use a certificate issued by a trusted certification authority.

To start, you need to generate a self-signed certificate and add it to Azure Key Vault. You can use the following guidance to perform this task.

[Acquire a self-signed certificate](https://docs.microsoft.com/en-us/azure/spring-cloud/expose-apps-gateway-end-to-end-tls?tabs=self-signed-cert%2Cself-signed-cert-2#acquire-a-certificate)

<details>
<summary>hint</summary>
<br/>

1. To create a self-signed certificate, you will use a sample-policy.json file. To generate the file, from the Git Bash shell prompt, run the following command:

   ```bash
   az keyvault certificate get-default-policy > sample-policy.json
   ```

1. From the Git Bash window, use your favorite text editor to open the sample-policy.json file, change its `subject` property and add the `subjectAlternativeNames` property to match the following content, save the file, and close it. 

   ```json
   {
     // ...
       "subject": "C=US, ST=WA, L=Redmond, O=Contoso, OU=Contoso HR, CN=myapp.mydomain.com",
       "subjectAlternativeNames": {
           "dnsNames": [
               "myapp.mydomain.com",
               "*.myapp.mydomain.com"
           ],
           "emails": [
               "hello@contoso.com"
           ],
           "upns": []
         },
     // ...
   }
   ```

   > **Note**: Ensure that you include the trailing comma at the end of the updated content as long as there is another JSON element following it.

1. Replace `mydomain` DNS name in the sample-policy.json file with a randomly generated custom domain name that you will use later in this exercise by running the following commands:

   ```bash
   DNS_LABEL=springclouddns$RANDOM$RANDOM
   DNS_NAME=sampleapp.${DNS_LABEL}.com
   cat sample-policy.json | sed "s/myapp.mydomain.com/${DNS_NAME}/g" > result-policy.json
   ```

1. Review the updated content of the result-policy.json file and record the updated DNS name in the format `sampleapp.<your-custom-domain-name>.com` (you will need it later in this exercise) by running the following command:

   ```bash
   cat result-policy.json
   ```

1. You can now use the **result-policy.json** file to create a self-signed certificate in Key Vault.

   ```bash
   CERT_NAME_IN_KV=openlab-certificate
   az keyvault certificate create \
       --vault-name $KEYVAULT_NAME \
       --name $CERT_NAME_IN_KV \
       --policy @result-policy.json
   ```

</details>

### Configure domain in Azure Spring Cloud

Now that you have a self-signed certificate added to the Azure Key Vault instance, as a next step you will configure a public domain name in Azure Spring Cloud using this self-signed certificate. You can use the following guidance to perform this task.

[Configure the public domain name on Azure Spring Cloud](https://docs.microsoft.com/en-us/azure/spring-cloud/expose-apps-gateway-end-to-end-tls?tabs=self-signed-cert%2Cself-signed-cert-2#configure-the-public-domain-name-on-azure-spring-cloud)

You will only create a custom domain for the api-gateway service. This is the only endpoint that you will expose externally.

<details>
<summary>hint</summary>
<br/>

1. To start, you need to provide Azure Spring Cloud the permissions to read the certificate from Key Vault. To accomplish this, you first need to identify the URI to your Key Vault and the object ID of Spring Cloud service.

   ```bash
   VAULTURI=$(az keyvault show -n $KEYVAULT_NAME -g $RESOURCE_GROUP --query properties.vaultUri -o tsv)
   ASCDM_OID=$(az ad sp show --id 03b39d0f-4213-4864-a245-b1476ec03169 --query objectId --output tsv)
   ```

1. Once you have the Key Vault URI and the Spring Cloud object ID, you can grant the permissions to access Key Vault certificates to the Spring Cloud service:

   ```bash
   az keyvault set-policy -g $RESOURCE_GROUP -n $KEYVAULT_NAME  --object-id $ASCDM_OID --certificate-permissions get list --secret-permissions get list
   ```

1. Next, configure TLS using the certificate.

   ```bash
   CERT_NAME_IN_ASC=openlab-certificate
   az spring-cloud certificate add \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_CLOUD_SERVICE \
       --name $CERT_NAME_IN_ASC \
       --vault-certificate-name $CERT_NAME_IN_KV \
       --vault-uri $VAULTURI
   ```

1. To conclude this procedure, you need to bind the custom domain to the api-gateway app.

   ```bash
   APPNAME=api-gateway
   az spring-cloud app custom-domain bind \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_CLOUD_SERVICE \
       --domain-name $DNS_NAME \
       --certificate $CERT_NAME_IN_ASC \
       --app $APPNAME
   ```

</details>

### Create the Application Gateway resources

You are now ready to create an Application Gateway instance to expose your application to the internet. You can use the following guidance to perform this task.

[Create the Application Gateway resources](https://docs.microsoft.com/en-us/azure/spring-cloud/expose-apps-gateway-end-to-end-tls?tabs=self-signed-cert%2Cself-signed-cert-2#create-network-resources)

<details>
<summary>hint</summary>
<br/>

   > **Note**: An Application Gateway resource needs a dedicated subnet to be deployed into, however, you already created this subnet at the beginning of this exercise. 

1. An Application Gateway instance also needs a public IP address, which you will create next by running the following commands from the Git Bash shell: 

   ```bash
   APPLICATION_GATEWAY_PUBLIC_IP_NAME=app-gw-openlab-public-ip
   az network public-ip create \
       --resource-group $RESOURCE_GROUP \
       --location $LOCATION \
       --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
       --allocation-method Static \
       --sku Standard \
       --dns-name $DNS_LABEL
   ```

1. In addition, an Application Gateway instance also needs to have access to the self-signed certificate in your Key Vault. To accomplish this, you will create a managed identity associated with the Application Gateway instance and retrieve the object ID of this identity.

   ```bash
   APPGW_IDENTITY_NAME=msi-appgw-openlab
   az identity create \
       --resource-group $RESOURCE_GROUP \
       --name $APPGW_IDENTITY_NAME

   APPGW_IDENTITY_CLIENTID=$(az identity show --resource-group $RESOURCE_GROUP --name $APPGW_IDENTITY_NAME --query clientId --output tsv)
   APPGW_IDENTITY_OID=$(az ad sp show --id $APPGW_IDENTITY_CLIENTID --query objectId --output tsv)
   ```

1. You can now reference the object ID when granting the get and list permissions to the Key Vault secrets and certificates.

   ```bash
   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --object-id $APPGW_IDENTITY_OID \
       --secret-permissions get list \
       --certificate-permissions get list
   ```

   > **Note**: In order for this implementation to work, the Application Gateway instance requires access to certificate and secrets in the Azure Key Vault instance.

1. Next, you need to retrieve the ID of the self-signed certificate stored in your Key Vault (you will use it in the next step of this task).

   ```bash
   KEYVAULT_SECRET_ID_FOR_CERT=$(az keyvault certificate show --name $CERT_NAME_IN_KV --vault-name $KEYVAULT_NAME --query sid --output tsv)
   ```

1. With all relevant information collected, you can now provision an instance of Application Gateway.

   ```bash
   APPGW_NAME=appgw-openlab
   APPNAME=api-gateway
   SPRING_APP_PRIVATE_FQDN=${APPNAME}.private.azuremicroservices.io

   az network application-gateway create \
       --name $APPGW_NAME \
       --resource-group $RESOURCE_GROUP \
       --location $LOCATION \
       --capacity 2 \
       --sku WAF_v2 \
       --frontend-port 443 \
       --http-settings-cookie-based-affinity Disabled \
       --http-settings-port 443 \
       --http-settings-protocol Https \
       --public-ip-address $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
       --vnet-name $VIRTUAL_NETWORK_NAME \
       --subnet $APPLICATION_GATEWAY_SUBNET_NAME \
       --servers $SPRING_APP_PRIVATE_FQDN \
       --key-vault-secret-id $KEYVAULT_SECRET_ID_FOR_CERT \
       --identity $APPGW_IDENTITY_NAME
   ```

1. To complete the configuration of the instance of Application Gateway, you need to retrieve the public key of the self-signed certificate, which is required to configure (in the next step) that certificate as issued by a trusted certification authority.

   ```bash
   az keyvault certificate download \
       --vault-name $KEYVAULT_NAME \
       --name $CERT_NAME_IN_KV \
       --file ./selfsignedcert.crt \
       --encoding DER

   az network application-gateway root-cert create \
       --resource-group $RESOURCE_GROUP \
       --cert-file ./selfsignedcert.crt \
       --gateway-name $APPGW_NAME \
       --name MySelfSignedTrustedRootCert
   ```

1. Finally, you can now update the HTTP settings of the Application Gateway instance to configure self-signed certificate as issued by a trusted certification authority.

   ```bash
   az network application-gateway http-settings update \
       --resource-group $RESOURCE_GROUP \
       --gateway-name $APPGW_NAME \
       --host-name-from-backend-pool false \
       --host-name $DNS_NAME \
       --name appGatewayBackendHttpSettings \
       --root-certs MySelfSignedTrustedRootCert
   ```

</details>

### Access the application by DNS name

You now have completed all steps required to test whether your application is accessible from the internet via Application Gateway. You can use the following guidance to perform this task.

[Check the deployment of Application Gateways](https://docs.microsoft.com/en-us/azure/spring-cloud/expose-apps-gateway-end-to-end-tls?tabs=self-signed-cert%2Cself-signed-cert-2#check-the-deployment-of-application-gateway)
[Configure DNS and access the application](https://docs.microsoft.com/en-us/azure/spring-cloud/expose-apps-gateway-end-to-end-tls?tabs=self-signed-cert%2Cself-signed-cert-2#configure-dns-and-access-the-application)

<details>
<summary>hint</summary>
<br/>

1. Check the back end health of the instance of Application Gateway you deployed in the previous task.

   ```bash
   az network application-gateway show-backend-health \
       --name $APPGW_NAME \
       --resource-group $RESOURCE_GROUP
   ```

   > **Note**: The output of this command should return the **Healthy** value of the **health** property of the **backendHttpSettingsCollection** element. If this is the case, your setup is valid. If you see any other value than healthy, review the previous steps.

   > **Note**: There might be a delay before the Application Gateway reports the **Healthy** status of **backendHttpSettingsCollection**, so if you encounter any issues, wait a few minutes and re-run the previous command before you start troubleshooting.

1. Next, identify the public IP address of the Application Gateway by running the following command from the Git Bash shell.

   ```bash
   az network public-ip show \
       --resource-group $RESOURCE_GROUP \
       --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
       --query [ipAddress] \
       --output tsv
   ```

1. To identify the custom DNS name associated with the certificate you used to configure the endpoint exposed by the Application Gateway instance, run the following command from the Git Bash shell.

   ```bash
   echo $DNS_NAME
   ```

   > **Note**: To validate the configuration, you will need to use the custom DNS name to access the public endpoint of the api-gateway app, exposed via the Application Gateway instance. You can test this by adding an entry that maps the DNS name to the IP address you identified in the previous step to the hosts file on your lab computer. 

1. On you lab computer, open the file **C:\Windows\System32\drivers\etc\hosts** in Notepad using elevated privileges (as administrator) and add an extra line to the file that has the following content (replace the `<app-gateway-ip-address> ` and `<custom-dns-name>` placeholders with the IP address and the DNS name you identified in the previous two steps):

   ```text
   <app-gateway-ip-address>   <custom-dns-name>
   ```

1. On your lab computer, start a web browser and, in the web browser window navigate to the URL that consists of the `https://` prefix followed by the custom DNS name you specified when updating the local hosts file. Your browser may display a warning notifying you that your connection is not private, but this is expected since you are relying on self-signed certificate. Acknowledge the warning but proceed to displaying the target web page. You should be able to see the PetClinic application start page again.

   > **Note**: While the connection to the MySQL database should be working at this point, keep in mind that this connectivity is established via a its public endpoint, rather than the private one. You will remediate this in the next exercise of this lab.

</details>

### Configure WAF on Application Gateway

Now that you have successfully deployed Application Gateway and you can connect to your application, you can additionally configure a Web Application Firewall on your Application Gateway. You can use the following guidance to perform this task.

[Create Web Application Firewall policies for Application Gateway](https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/create-waf-policy-ag)
[az network application-gateway waf-policy](https://docs.microsoft.com/en-us/cli/azure/network/application-gateway/waf-policy?view=azure-cli-latest)
[az network application-gateway http-listener](https://docs.microsoft.com/en-us/cli/azure/network/application-gateway/http-listener?view=azure-cli-latest)

<details>
<summary>hint</summary>
<br/>

1. Create a new WAF policy. By default a WAF policy has the OWASP ruleset associated with it.

   ```bash
   WAF_POLICY_NAME=openlabWAFPolicy
   az network application-gateway waf-policy create \
       --name $WAF_POLICY_NAME \
       --resource-group $RESOURCE_GROUP
   ```

1. You can now link this WAF policy to the HTTP listener for the incoming requests.

   ```bash
   az network application-gateway http-listener list \
       -g $RESOURCE_GROUP \
       --gateway-name $APPGW_NAME -o table

   az network application-gateway http-listener update \
       -g $RESOURCE_GROUP \
       --gateway-name $APPGW_NAME \
       -n appGatewayHttpListener \
       --waf-policy $WAF_POLICY_NAME
   ```

1. To conclude the setup, enable the WAF policy. This will automatically start flagging noncompliant requests. To avoid blocking any requests at this point, configure it in detection mode.

   ```bash
   az network application-gateway waf-policy policy-setting update \
       --mode Detection \
       --policy-name $WAF_POLICY_NAME \
       --resource-group $RESOURCE_GROUP \
       --state Enabled
   ```

</details>

#### Review

In this lab, you enhanced network security of Azure Spring Cloud apps by blocking connections to its public endpoints and adding a Web Application Firewall to filter incoming requests. 
