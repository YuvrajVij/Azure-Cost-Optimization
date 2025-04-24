# Azure-Cost-Optimization

---
# Introduction
This will provide a high-level idea by which we can optimize our Cost on the Azure portal. Since Azure Cosmos DB consumes a lot of storage as well as cost, so in order to optimize it, we can migrate our 90-day back records into an Azure Blob Storage container, which is comparatively cheaper than Azure Cosmos DB.

# High‑Level Architecture

📊 Architecture Overview

          +-----------------+
          |   Billing API   |
          +-----------------+
                   |
                   |
                   ▼
        +---------------------------------------+
        |   Cosmos DB (90-day hot data only)    |
        +---------------------------------------+
                ⏱️ Daily Schedule
                ▼
                +-----------------------------+
                |   Azure DevOps Pipeline     |
                +-----------------------------+
                                |
                                |
                                ▼
                +-----------------------------------------------------------------------+
                |   Run Python Script:                                                  |
                |   1. Query Cosmos for records > 3 months                             |
                |   2. Archive to Blob (Cool/Archive tier)                             |
                |   3. (Optional) Delete from Cosmos if archived                       |
                +-----------------------------------------------------------------------+
                                                        |
                                                        |
                                                        ▼
                          +----------------------+         +-----------------------+
                          |   Azure Blob Storage |<--------|   Data Retrieval Path  |
                          |  (Cool/Archive Tier) |         +-----------------------+
                          +----------------------+                          |
                                                                            |
                                                                            |
                                                                            ▼
                                                                    +-----------------+
                                                                    |   Billing API   |
                                                                    +-----------------+



---


# 1. Provision Resources
 1. Cosmos DB

- Ensure you have a Cosmos SQL API account and your “billing” container.

- Enable Change Feed (default on SQL API).

 2. Azure Storage Account

- Create a Storage Account.

- Create a Blob Container (e.g. billing-archive), set default access tier to Cool or Archive.

3. Azure Function App

- Create a Function App (Python runtime).

- Assign it a Managed Identity, then grant “Storage Blob Data Contributor” on the storage account.

- Assign it “Cosmos DB Built‑in Data Reader” on the Cosmos account.


---

2. Configure TTL on Cosmos Container
In the Azure Portal, under your billing container’s Settings → Time to Live, set:

```
Enabled: Yes
Default TTL (seconds): 7776000   # 90 days × 24h × 3600s

```
This auto‑deletes documents older than 3 months, but only after they have been archived.


---

3. Azure Function: Change‑Feed to Blob
Create a Function named ArchiveBillingRecords in your Function App. Use the Cosmos DB trigger. Below is a simplified function_app.py.



```
import logging
import azure.functions as func
import os
import json
from datetime import datetime
from azure.storage.blob import BlobServiceClient

# Blob Storage setup via Managed Identity
BLOB_ACCOUNT_URL = os.getenv("BLOB_ACCOUNT_URL")  # e.g. https://<storage>.blob.core.windows.net
CONTAINER_NAME    = os.getenv("BLOB_CONTAINER", "billing-archive")
blob_service = BlobServiceClient(account_url=BLOB_ACCOUNT_URL, credential=None)
container_client = blob_service.get_container_client(CONTAINER_NAME)

def main(documents: func.DocumentList) -> None:
    """
    Triggered by new or updated items in Cosmos DB change feed.
    Archives each record as a JSON blob under YYYY/MM/<recordId>.json
    """
    if not documents:
        logging.info("No documents in feed.")
        return

    for doc in documents:
        try:
            # Assuming each doc has 'id' and 'timestamp' fields
            record_id = doc.get("id")
            ts        = doc.get("timestamp")  # ISO 8601 string
            dt        = datetime.fromisoformat(ts.rstrip("Z"))
            path      = f"{dt.year}/{dt.month:02d}/{record_id}.json"

            blob_client = container_client.get_blob_client(path)
            blob_client.upload_blob(
                data=json.dumps(doc, default=str),
                overwrite=True
            )
            logging.info(f"Archived record {record_id} → {path}")
        except Exception as e:
            logging.error(f"Error archiving doc {doc}: {e}")


```

---

Function Configuration
In function.json:

```
{
  "bindings": [
    {
      "name": "documents",
      "type": "cosmosDBTrigger",
      "direction": "in",
      "leaseCollectionName": "leases",
      "connectionStringSetting": "COSMOS_DB_CONN",
      "databaseName": "<your-db-name>",
      "collectionName": "<your-billing-container>",
      "createLeaseCollectionIfNotExists": true
    }
  ]
}


```

In your Function App Application Settings, add:

- COSMOS_DB_CONN → your Cosmos DB connection string

- BLOB_ACCOUNT_URL → e.g. https://mystorage.blob.core.windows.net

- BLOB_CONTAINER → billing-archive

---


4.  DevOps pipeline Implementation

4.1.  Write Archival Python Script (archive_cosmos_to_blob.py)

```

import os
import json
from datetime import datetime, timedelta
from azure.cosmos import CosmosClient, PartitionKey
from azure.storage.blob import BlobServiceClient

# === Config ===
COSMOS_URL    = os.environ["COSMOS_URL"]
COSMOS_KEY    = os.environ["COSMOS_KEY"]
DB_NAME       = os.environ["COSMOS_DB"]
CONTAINER     = os.environ["COSMOS_CONTAINER"]
BLOB_ACCOUNT  = os.environ["BLOB_ACCOUNT_URL"]
BLOB_CONTAINER = os.environ["BLOB_CONTAINER"]

# Connect to Cosmos DB
cosmos = CosmosClient(COSMOS_URL, credential=COSMOS_KEY)
db     = cosmos.get_database_client(DB_NAME)
container = db.get_container_client(CONTAINER)

# Connect to Blob
blob_service = BlobServiceClient(account_url=BLOB_ACCOUNT, credential=None)
blob_container = blob_service.get_container_client(BLOB_CONTAINER)

# === Archive logic ===
cutoff_date = datetime.utcnow() - timedelta(days=90)

query = f"SELECT * FROM c WHERE c.timestamp < '{cutoff_date.isoformat()}'"
old_records = list(container.query_items(query, enable_cross_partition_query=True))

print(f"Found {len(old_records)} records older than 90 days")

for record in old_records:
    record_id = record.get("id")
    ts = record.get("timestamp")
    dt = datetime.fromisoformat(ts.rstrip("Z"))
    blob_path = f"{dt.year}/{dt.month:02d}/{record_id}.json"

    blob_client = blob_container.get_blob_client(blob_path)

    blob_client.upload_blob(
        json.dumps(record, default=str),
        overwrite=True
    )
    print(f"Archived: {blob_path}")

    # Optional: Delete from Cosmos if successfully archived
    try:
        container.delete_item(record_id, partition_key=record_id)
        print(f"Deleted from Cosmos: {record_id}")
    except Exception as e:
        print(f"Error deleting {record_id}: {e}")

```
4.2. Store in Repo (e.g., infra/archival/ directory)
Create the following folder structure in your repo:

```
/infra
  /archival
    archive_cosmos_to_blob.py

```


4.3. Azure DevOps Pipeline YAML
Add this pipeline to your repo under .azure-pipelines/archive-cosmos-blob.yml:


```
trigger: none  # Manual or scheduled

schedules:
  - cron: "0 2 * * *"  # Daily at 2 AM UTC
    displayName: Daily Cosmos Archival
    branches:
      include:
        - main
    always: true

pool:
  vmImage: ubuntu-latest

variables:
  COSMOS_URL: "<your-cosmos-uri>"
  COSMOS_KEY: "$(COSMOS_KEY)"          # stored securely in pipeline
  COSMOS_DB: "<your-db>"
  COSMOS_CONTAINER: "<your-container>"
  BLOB_ACCOUNT_URL: "<your-blob-url>"
  BLOB_CONTAINER: "billing-archive"

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.11'

  - script: |
      pip install azure-cosmos azure-storage-blob
      python infra/archival/archive_cosmos_to_blob.py
    env:
      COSMOS_URL: $(COSMOS_URL)
      COSMOS_KEY: $(COSMOS_KEY)
      COSMOS_DB: $(COSMOS_DB)
      COSMOS_CONTAINER: $(COSMOS_CONTAINER)
      BLOB_ACCOUNT_URL: $(BLOB_ACCOUNT_URL)
      BLOB_CONTAINER: $(BLOB_CONTAINER)
    displayName: Archive old billing records


```

4.4. Secure Your Secrets in Azure DevOps
Go to Project → Pipelines → Library and create a Variable Group named cosmos-archive-secrets.
Add:

- COSMOS_KEY as a secret

- COSMOS_URL, BLOB_ACCOUNT_URL, COSMOS_DB, COSMOS_CONTAINER, etc.

Then link the variable group to your pipeline.


---
✅ Benefits

| Feature |	✅ Covered? | 
|--|--|
| Simplicity	| No code changes in API
| No Data Loss | Only delete after archive
| No Downtime  | Scheduled background task
| API Contract Unchanged | 	Yes, nothing changes externally
| Cost Optimization 	| Cosmos purged, Blob is cheap
