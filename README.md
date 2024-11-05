## Requirements 
1. **Azure Account:** If you don't already have an Azure subscription, create a free account using the following link: https://azure.microsoft.com/en-gb/free/.
2. **Azure Cloud Shell:** You can launch and run the free interactive shell in the Azure portal. It has common Azure tools preinstalled and configured to use with your account.
3. **Azure CLI:** Alternatively, you can opt to use the terminal on your local machine. Ensure that Azure CLI is installed on your machine. Follow the official guidance on [How to Install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

I will be using the terminal on my local machine for this task. 

## Prepare Your Environment 
1. Login to the Azure portal and authenticate your account via the terminal by running the following command:

    ```
    az login
    ```

You will not be able to run commands in Azure using the CLI without completing the above step. 

### Create Storage Account 
Before we can proceed with [....] , we must create a resource group with the ```az group create``` command. When we create, provision, deploy etc. a resource, it must be assigned to a resource group. Resource groups are used to organise resources in Azure. Think of a resource group as a folder that helps you organise and manage your resources efficiently. 

2. Run the following command to create a resource group named **rg-VNpeering-001** in **westeurope**. 

   ```
   az group create --name rg-StaticWebsite-001 --location northeurope
   ```

   Next, we need to create a storage account which we will use to store...

   ```
    az storage account create \
    --name staticwebstorage001 \
    --resource-group rg-StaticWebsite-001 \
    --location northeurope \
    --sku Standard_LRS
   ```

Enable static website hosting

  ```
  az storage blob service-properties update \
  --account-name staticwebstorage001 \
  --static-website \
  --404-document 404.html \
  --index-document index.html \
  --auth-mode login
  ```

Note: The above command will create a container called $web in your blob storage container that will enable you to upload your `index.html` and `404.html` files. You can use the Azure Portal or CLI to upload your website. This example will use Azure CLI. Run the following command: 

  ```
  az storage blob upload-batch \
  --account-name staticwebstorage001 \
  --source <source-path> \
  --destination '$web' 
  ```

You can find the public URL of your static website by using the following command:

  ```
  az storage account show \
  --name staticwebstorage001 \
  --resource-group rg-StaticWebsite-001 \
  --query "primaryEndpoints.web" \
  --output tsv
  ```

You should see output similar to the following: 

```
https://staticwebstorage001.z16.web.core.windows.net/
```

## Custom Domain

### Create an Azure Content Delivery Network (CDN)

...

Run the following command 

  ```
  az cdn profile create \
  --name StaticWebsiteCDN \
  --resource-group rg-StaticWebsite-001 \
  --sku Standard_Microsoft
  ```

Create the CDN endpoint and set the origin to your storage URL:

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

### Add a Custom Domain to Your CDN

Find a domain registrar of choice to purchase and register a custom domain. I will be using GoDaddy.com for this example. If you already have a custom domain, skip this step. 

You can use a subdomain or root domain for your website. If you want to use a subdomain, simply create a CNAME DNS record in your chosen domain registrar that points from your custom domain to your Azure CDN endpoint. I entered the following for my subdomain: 

```
TYPE: CNAME
NAME: www.
VALUE: StaticWebsiteCDNEndpoint001.azureedge.net
TTL: Leave default or select 1 hour. 
```

It's worth noting that it may take a while before the CNAME is fully propogated across all DNS servers, and you can proceed with add a custom domain to your CDN. Check in 30-60 minutes. Propagation usually completes within a few hours with most providers.

It can be helpful to use a website like <DNS checker> website and/or running the `nslookup` command to check the progress of DNS propagation. I found using a DNS tracker especially helpful for identifying whether I had any potential configuration issues, and for verifying when the process was complete atfer an hour. 

```
nslookup www.miticamatthews.com
```

Next, you need to add a custom domain to your CDN. To do this, run the following command: 

  ```
  az cdn custom-domain create \
  --resource-group rg-StaticWebsite-001 \
  --profile-name StaticWebsiteCDN \
  --endpoint-name StaticWebsiteCDNEndpoint001 \
  --hostname www.miticamatthews.com \
  --name MMPortfolioSite
  ```

### Enable HTTPS for Subdomains

To enable HTTPS for your custom subdomain, run the following command: 

  ```
  az cdn custom-domain enable-https \
  --endpoint-name StaticWebsiteCDNEndpoint001 \
  --name MMPortfolioSite \
  --profile-name StaticWebsiteCDN \
  --resource-group rg-StaticWebsite-001 
  ```

Azure will automatically verify your custom domain and create a certificate for it using the CNAME record you set up in your DNS registrar, which points to the CDN endpoint hostname you created earlier. This process does not require any approval from you. However, if the CNAME record no longer exists or Azure cannot locate it, an email will be sent to your domain's admin email address for verification, and you will need to confirm domain ownership. Once verified, Azure will issue the certificate, which is valid for one year and will renew automatically before it expires. 

Automatic validation usually takes a few hours, and you can monitor the progress by checking your CDN endpoint custom domain in the Azure Portal.
