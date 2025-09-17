# Azure File Setup Guide

This guide walks you through creating and configuring Azure File shares
with Terraform (TF) and the Azure portal, including optional integration
with Active Directory (AD) and Azure File Sync.

------------------------------------------------------------------------

## 1. Prerequisites

-   **Azure Subscription** with required permissions.
-   **Terraform** installed and configured.
-   **PowerShell** with Az modules installed.
-   **Access to an AD environment** (if enabling AD integration).
-   Optional: **Azure Storage Explorer** for managing files.

------------------------------------------------------------------------

## 2. Create a Resource Group

Login to Azure:

``` bash
az login
```

Terraform configuration:

``` hcl
resource "azurerm_resource_group" "rg" {
  name     = "name-of-new-rg"   # Change to your desired resource group name
  location = "West US"          # Change to your desired region

  tags = {
    Owner       = "First Last"       # Replace with actual owner
    Category    = "Storage"          # Replace with category
    Description = "Azure File Storage"
  }
}
```

------------------------------------------------------------------------

## 3. Create a Storage Account

Import resource group:

``` bash
terraform import azurerm_resource_group.rg /subscriptions/<subscription-id>/resourceGroups/name-of-new-rg
```

Terraform configuration:

``` hcl
resource "azurerm_storage_account" "nameofstorageaccount" {
  name                     = "nameofstorageaccount" # Must be globally unique, lowercase, no special chars
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  network_rules {
    default_action = "Deny"       # Deny all traffic by default
    ip_rules       = ["x.x.x.x"]  # Replace with allowed IP(s)
  }

  tags = {
    Owner       = "First Last"
    Category    = "Storage"
    Description = "Azure File Storage Account"
  }
}
```

------------------------------------------------------------------------

## 4. Create a File Share (Portal)

1.  Go to **[portal.azure.com](https://portal.azure.com)**.\
2.  Navigate to your **Storage Account** → **Data Storage** → **File
    Shares**.\
3.  Select **+ File Share**.\
4.  **Configure File Share**:
    -   **Name** the share.\
    -   Select **Access Tier = Hot** from the drop-down menu.\
    -   *Note:* The name of the share will serve as the root folder but
        will not appear directly on the file server.\
5.  **Configure Backup**:
    -   Select **Backup**.\
    -   Ensure **Enable Backup** is checked.\
    -   **Name the Recovery Services Vault** (or select an existing
        one).\
6.  **Edit Backup Policy**:
    -   Select **Edit Policy**.\
    -   Update the policy **name** (e.g., `FileShareDailyBackup`).\
    -   Modify **frequency, time, timezone, and retention** per
        requirements.
        -   Example: Daily at **7:30 PM UTC**, retained for **30
            days**.\
7.  Select **Review + Create**.\
8.  Ensure validation passes, then select **Create**.

------------------------------------------------------------------------

## 5. Deploy Azure File Sync Service

1.  From the resource group → **Create Resource**.\
2.  Search for **Azure File Sync** → Create.\
3.  Provide a name.\
4.  Networking → Allow access from all networks.\
5.  Add Tags if needed.\
6.  Review + Create.

------------------------------------------------------------------------

## 6. Create a Sync Group

1.  Open the **Sync Service**.\
2.  Create a **Sync Group**.\
3.  Provide a name.\
4.  Select your **Storage Account** and the **File Share** created
    earlier.\
5.  Create.

------------------------------------------------------------------------

## 7. Join Storage Account to Active Directory (Optional)

Install required modules:

``` powershell
Install-Module -Name Az.Storage -Force -AllowClobber
Import-Module Az.Storage
Install-Module -Name AzFilesHybrid -Scope AllUsers
```

Login and join:

``` powershell
Connect-AzAccount

Join-AzStorageAccount `
  -ResourceGroupName "name-of-new-rg" `
  -StorageAccountName "nameofstorageaccount" `
  -Domain "domain.com" `
  -DomainAccountType ComputerAccount `
  -OrganizationalUnitDistinguishedName "OU=_StorageAccounts,DC=domain,DC=com"
```

The storage account should now appear as a computer object in AD.

------------------------------------------------------------------------

## 8. Provision an Azure File Server

Using Terraform: 
1. Create a VM.\
2. Attach a data disk (e.g., `D:\Data`).\
3. Install **Azure File Sync Agent**:\
- [Download
here](https://www.microsoft.com/en-us/download/details.aspx?id=57159)\
- Run through setup wizard.\
- Register server with Azure resources created above.

------------------------------------------------------------------------

## 9. Configure Sync Endpoints

1.  Open the **Sync Group** created earlier.\
2.  Add **Server Endpoint**:
    -   Choose the registered server.\
    -   Path (e.g., `D:\FileShare`).\
    -   Enable Cloud Tiering.\
    -   Set volume free space policy (e.g., 60%).\
    -   Configure date policy (e.g., 90 days).\
    -   Choose initial sync method: *Namespace only* or *Merge content*.

------------------------------------------------------------------------

## 10. Upload Data via Azure Storage Explorer

1.  [Download Storage
    Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer#Download-4).\
2.  Sign in with Azure credentials.\
3.  Navigate to your **Storage Account** → **File Share**.\
4.  Upload data (files/folders).

------------------------------------------------------------------------
