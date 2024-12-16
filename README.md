## Requirements 
1. **Azure Account:** If you don't already have an Azure subscription, create a free account using the following link: https://azure.microsoft.com/en-gb/free/.
2. **Azure Cloud Shell:** You can launch and run the free interactive shell in the Azure portal. It has common Azure tools preinstalled and configured to use with your account.
3. **Azure CLI:** Alternatively, you can opt to use the terminal on your local machine. Ensure that Azure CLI is installed on your machine. Follow the official guidance on [How to Install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

I will be using the terminal on my local machine for this task. 

## Setting Up the AZ CLI Environment

Once AZ CLI is installed, run the following command to log in to your Azure account. You will not be able to run commands in Azure using the CLI without completing the above step:

 ```
    az login
 ```
A browser window will pop up, prompting you to login. Once this step is complete, the browser window will close and az cli will display a list of subscriptions to choose from that you wish to charge your resources to. Ensure you select the right one.

```
az account set --subscription <YourSubscriptionID>
```
 

## Create a Resource Group

You will need a resource group to simplify the management of all the resources used in this project, such as Azure Storage and Azure CDN resources. Create a resource group if you don't already have one. 

The following command creates a resource group named **rg-StaticWebsite-001** in **northeurope**. 

   ```
   az group create --name rg-StaticWebsite-001 --location northeurope
   ```

## Create a Storage Account

Next, create a storage account:

   ```
    az storage account create \
    --name staticwebstorage001 \
    --resource-group rg-StaticWebsite-001 \
    --location northeurope \
    --sku Standard_LRS
   ```

Then, enable static website hosting for this account: 

  ```
  az storage blob service-properties update \
  --account-name staticwebstorage001 \
  --static-website \
  --404-document 404.html \
  --index-document index.html \
  --auth-mode login
  ```
The command creates a blob container called $web in your storage account, allowing you to upload files like index.html and 404.html for public access, available via the URL `<accountname>.z16.web.core.windows.net/`. For example, `staticwebstorage001.z16.web.core.windows.net`.

You can also find the public URL of your static website by using the following command:

  ```
  az storage account show \
  --name staticwebstorage001 \
  --resource-group rg-StaticWebsite-001 \
  --query "primaryEndpoints.web" \
  --output tsv
  ```

## Deploy your Website 

Next, use Azure CLI to upload the contents of the relevant local directory to blob storage using the following command:

  ```
  az storage blob upload-batch \
  --account-name staticwebstorage001 \
  --source <Source-Path> \
  --destination '$web' 
  ```


## Custom Domain

### Create an Azure Content Delivery Network (CDN) Profile & Endpoint

Next, we’ll set up a Content Delivery Network (CDN) to optimize content delivery by caching site content on globally distributed servers, which improves performance and reduces load times for users. Setting up a CDN also allows us to configure SSL on a custom domain name, adding an extra layer of security. This setup involves creating two components: a CDN Profile to manage the CDN type and configuration, and an Azure CDN Endpoint to connect our content to the CDN. We’ll use the "Standard Microsoft CDN" for this walkthrough, as it offers a solid balance of features and pricing for most use cases.

Run the following command to create a CDN Profile:

  ```
  az cdn profile create \
  --name StaticWebsiteCDN \
  --resource-group rg-StaticWebsite-001 \
  --sku Standard_Microsoft
  ```

Next, we will create the CDN Endpoint. We need to set the origin to the storage (static hosting) URL from the previous step:

  ```
  az cdn endpoint create \
  --name StaticWebsiteCDNEndpoint001 \
  --resource-group rg-StaticWebsite-001 \
  --profile-name StaticWebsiteCDN \
  --origin staticwebstorage001.z16.web.core.windows.net \
  --origin-host-header staticwebstorage001.z16.web.core.windows.net
  ```

To get the `hostname` URL for your Azure CDN endpoint, run the following command: 
 
  ```
az cdn endpoint show \
  --resource-group rg-StaticWebsite-001 \
  --profile-name StaticWebsiteCDN \
  --name StaticWebsiteCDNEndpoint001 \
  --query "hostName" \
  --output tsv
  ```

## Configuring a Domain with an Azure Managed Certificate

The final step in setting up the CDN Endpoint is to associate it with a custom domain and enable HTTPS. Associating a custom domain with the CDN Endpoint allows users to access the website using a memorable, user-friendly URL. This step involves creating a CNAME (Canonical Name) record with your DNS registrar to map the domain to the CDN Endpoint, ensuring it connects smoothly to your hosted content. In this example, we will be using GoDaddy as the DNS provider. 

1. Choose a DNS registrar (e.g., GoDaddy) and register a custom domain. 
2. Decide whether you will use a root domain or subdomain for your site. Here, we will use a subdomain. 
3. Create a CNAME record with the following settings: 

```
TYPE: CNAME
NAME: www.
VALUE: StaticWebsiteCDNEndpoint001.azureedge.net
TTL: Leave default or select 1 hour. 
```
You can use the full custom domain e.g., **www.miticamatthews.com** for the name entry. However, if you have any problems, use `www.` instead as I have above. 

It's worth noting that DNS propagation may take some time to fully complete across all DNS servers before the custom domain can be added to the CDN. Typically, propagation takes a few hours, but it’s a good idea to start checking after 30–60 minutes.

To monitor progress, tools like [DNS Checker](https://dnschecker.org/) or the `nslookup` command can be useful for verifying the CNAME record status. I found using a DNS tracker especially useful for identifying any configuration issues early on and for verifying when the propagation was successful and complete. 

```
nslookup www.miticamatthews.com
```

### Create a Custom Domain for the CDN Endpoint 

Once the CNAME record is set up, use the following command to register your custom domain with the CDN endpoint: 
  ```
  az cdn custom-domain create \
  --resource-group rg-StaticWebsite-001 \
  --profile-name StaticWebsiteCDN \
  --endpoint-name StaticWebsiteCDNEndpoint001 \
  --hostname www.miticamatthews.com \
  --name MMPortfolioSite
  ```

### Enable HTTPS for Subdomains

Finally, run the following command to enable HTTPS, securing the connection between users and the website: 

  ```
  az cdn custom-domain enable-https \
  --endpoint-name StaticWebsiteCDNEndpoint001 \
  --name MMPortfolioSite \
  --profile-name StaticWebsiteCDN \
  --resource-group rg-StaticWebsite-001 
  ```

Azure automatically verifies your custom domain and create a certificate using the CNAME record you set up in your DNS registrar, which points to your CDN Endpoint. No approval is required unless Azure cannot find the CNAME record, in which case an email will be sent to your domain's admin for verification. Once verified, Azure will issue a certificate that is valid for one year and will auto-renew before it expires. 

With Azure CDN, HTTPS is automatically configured with an Azure-managed certificate, so there’s no need for third-party SSL management. The certificate will automatically renew, ensuring continuous security.

Validation tpically takes a few hours, and you can monitor the progress by checking your CDN Endpoint custom domain in the Azure Portal.

<img width="738" alt="CDN Endpoint Custom Domain Progress" src="https://github.com/user-attachments/assets/ff50f6a4-f7e8-42b0-b2b8-a87d9fe48cb2">

## Create CDN Endpoint Rules

Next, we need to add the following URL redirect rule to the CDN Endpoint created earlier to redirect all HTTP requests to HTTPS: 

```
az cdn endpoint rule add \
--name StaticWebsiteCDNEndpoint001 \
--resource-group rg-StaticWebsite-001 \
--profile-name StaticWebsiteCDN \
--rule-name enforceHTTPS \
--action-name UrlRedirect \
--order 1 \
--operator Equal \
--match-variable RequestScheme \
--match-values HTTP \
--redirect-protocol HTTPS \
--redirect-type Moved
 ```

## Deploy Website Updates

When using a CDN, it's important to remember that the cache doesn't automatically update when you upload new files to your storage account. The CDN will continue serving the cached versions until you explicitly instruct it to refresh its cache and pull in the updated files.

To do this, you can use `--content-paths '/'` to purge all content from the CDN cache. However, if you're updating specific files, it's more efficient to specify the exact file, e.g., `--content-paths '/index.html'` to avoid unnecessary purging and reduce potential increased load times when the cache is refreshed.

```
az cdn endpoint purge \
--resource-group rg-StaticWebsite-001 \
--name StaticWebsiteCDNEndpoint001 \
--profile-name StaticWebsiteCDN \
--no-wait \
--content-paths '/index.html'

```

The purge command can take some time to complete. To avoid having to wait for the process to finish, use the `--no-wait` option so that the command returns immediately while the purge continues in the background.

## Configure CosmosDB to Store Visitor Count


Run the following command to check if the provider is already registered: 

```
az provider show --namespace Microsoft.DocumentDB --query "registrationState"
```

If the output shows `"registrationState": "NotRegistered"`, run the following command to register the provider:

```
az provider register --namespace Microsoft.DocumentDB
```

Next, re-run the `az provider show` command that is two steps above, to ensure that the registration states is *"Registered"*. 

Run the following command to create a new Azure CosmosDB account:

```
az cosmosdb create \
--name portfolioweb-cdb-001 \
--resource-group rg-StaticWebsite-001 \
--kind GlobalDocumentDB \
--locations regionName=northeurope \
--capabilities EnableServerless \
--enable-free-tier true
```

Next, we need to create a database and container to store the visitor count. Run the following commands: 

```
az cosmosdb sql database create \
--account-name portfolioweb-cdb-001 \
--resource-group rg-StaticWebsite-001 \
--name VisitorCounterDB
```

```
az cosmosdb sql container create \
--account-name portfolioweb-cdb-001 \
--resource-group rg-StaticWebsite-001 \
--database-name VisitorCounterDB \
--name VisitorsContainer \
--partition-key-path /visitorId
```

