# velero-demo

```bash
#!/bin/bash

set -e  # Exit immediately if a command fails

# Set variables
AZURE_BACKUP_RESOURCE_GROUP="Velero_Backups"
AZURE_STORAGE_ACCOUNT_ID="dcgbbvelero"  # Change this to your desired storage account name
BLOB_CONTAINER="velero"
CUSTOM_ROLE_NAME="VeleroBackupRole"

# Azure region
AZURE_REGION="WestUS"

echo "Setting up Resource Group..."
az group create -n "$AZURE_BACKUP_RESOURCE_GROUP" --location "$AZURE_REGION"

echo "Creating Storage Account..."
az storage account create \
    --name "$AZURE_STORAGE_ACCOUNT_ID" \
    --resource-group "$AZURE_BACKUP_RESOURCE_GROUP" \
    --sku Standard_GRS \
    --encryption-services blob \
    --https-only true \
    --kind BlobStorage \
    --access-tier Hot

echo "Creating Blob Container..."
az storage container create -n "$BLOB_CONTAINER" --public-access off --account-name "$AZURE_STORAGE_ACCOUNT_ID"

echo "Retrieving Subscription and Tenant ID..."
AZURE_SUBSCRIPTION_ID=$(az account show --query 'id' -o tsv)
AZURE_TENANT_ID=$(az account show --query 'tenantId' -o tsv)

echo "Creating Service Principal for Velero..."
AZURE_CLIENT_SECRET=$(az ad sp create-for-rbac --name "velero" --role "Contributor" --query 'password' -o tsv --scopes "/subscriptions/$AZURE_SUBSCRIPTION_ID")

# Wait for service principal to propagate
sleep 10

AZURE_CLIENT_ID=$(az ad sp list --display-name "velero" --query '[0].appId' -o tsv)

echo "Creating Azure Role JSON..."
cat <<EOF > azure-role.json
{
    "Name": "$CUSTOM_ROLE_NAME",
    "Id": null,
    "IsCustom": true,
    "Description": "Velero related permissions to perform backups, restores, and deletions",
    "Actions": [
        "Microsoft.Compute/disks/read",
        "Microsoft.Compute/disks/write",
        "Microsoft.Compute/disks/endGetAccess/action",
        "Microsoft.Compute/disks/beginGetAccess/action",
        "Microsoft.Compute/snapshots/read",
        "Microsoft.Compute/snapshots/write",
        "Microsoft.Compute/snapshots/delete",
        "Microsoft.Storage/storageAccounts/listkeys/action",
        "Microsoft.Storage/storageAccounts/regeneratekey/action",
        "Microsoft.Storage/storageAccounts/read"
    ],
    "NotActions": [],
    "AssignableScopes": [
        "/subscriptions/$AZURE_SUBSCRIPTION_ID"
    ]
}
EOF

echo "Assigning Custom Role..."
az role definition create --role-definition azure-role.json

echo "Generating Velero credentials file..."
cat <<EOF > credentials-velero.txt
AZURE_SUBSCRIPTION_ID=${AZURE_SUBSCRIPTION_ID}
AZURE_TENANT_ID=${AZURE_TENANT_ID}
AZURE_CLIENT_ID=${AZURE_CLIENT_ID}
AZURE_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
AZURE_RESOURCE_GROUP=${AZURE_BACKUP_RESOURCE_GROUP}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

echo "Installing Velero..."
velero install \
    --provider azure \
    --use-node-agent \
    --namespace velero \
    --default-volumes-to-fs-backup \
    --bucket "$BLOB_CONTAINER" \
    --secret-file /home/dcasati/src/homelab/velero/credentials-velero.txt \
    --plugins velero/velero-plugin-for-microsoft-azure:v1.5.0 \
    --backup-location-config resourceGroup="$AZURE_BACKUP_RESOURCE_GROUP",storageAccount="$AZURE_STORAGE_ACCOUNT_ID",subscriptionId="$AZURE_SUBSCRIPTION_ID" \
    --features=EnableCSI \
    --uploader-type=kopia

echo "Velero setup completed successfully!"
```

Next steps:

1. Check if the backup location has been successfully configured by running the following command:
velero backup-location get

1. Create a backup for the pets namespace by running the following command:
velero backup create pets-backup --include-namespaces pets

1. deploy the nginx with pv example and create a backup that includes persistent volumes by running the following command:
velero backup create nxing-with-pv-backup --include-namespaces nginx-example --snapshot-volumes

1. Add completion to velero commands by running the following command:
source <(velero completion bash)

1. backup using kopia
velero backup create  nfs-backup-snapshot-2 --include-namespaces=default,nfs-server --snapshot-volumes=true --snapshot-move-data=true --default-volumes-to-fs-backup=true
