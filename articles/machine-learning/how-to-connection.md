---
title: Create connections to external data sources (preview)
titleSuffix: Azure Machine Learning
description: Learn how to use connections to connect to External data sources for training with Azure Machine Learning.
services: machine-learning
ms.service: azure-machine-learning
ms.subservice: mldata
ms.topic: how-to
ms.author: franksolomon
author: fbsolo-ms1
ms.reviewer: ambadal
ms.date: 07/24/2024
ms.custom: data4ml, devx-track-azurecli
# Customer intent: As an experienced data scientist with Python skills, I have data located in external sources outside of Azure. I need to make that data available to the Azure Machine Learning platform, to train my machine learning models.
---

# Create connections (preview)

[!INCLUDE [dev v2](includes/machine-learning-dev-v2.md)]

In this article, learn how to connect to data sources located outside of Azure, to make that data available to Azure Machine Learning services. Azure connections serve as key vault proxies, and interactions with connections are direct interactions with an Azure key vault. An Azure Machine Learning connection securely stores username and password data resources, as secrets, in a key vault. The key vault RBAC controls access to these data resources. For this data availability, Azure supports connections to these external sources:

- Snowflake DB
- Amazon S3
- Azure SQL DB

[!INCLUDE [machine-learning-preview-generic-disclaimer](includes/machine-learning-preview-generic-disclaimer.md)]

## Prerequisites

- An Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/).

- The [Azure Machine Learning SDK for Python](https://aka.ms/sdk-v2-install).

- An Azure Machine Learning workspace.

> [!IMPORTANT]
> An Azure Machine Learning connection securely stores the credentials passed during connection creation in the Workspace Azure Key Vault. A connection references the credentials from the key vault storage location for further use. You don't need to directly deal with the credentials after they are stored in the key vault. You have the option to store the credentials in the YAML file. A CLI command or SDK can override them. We recommend that you **avoid** credential storage in a YAML file, because a security breach could lead to a credential leak.

> [!NOTE]
> For a successful data import, please verify that you installed the latest **azure-ai-ml** package (version 1.5.0 or later) for SDK, and the ml extension (version 2.15.1 or later).
>
> If you have an older SDK package or CLI extension, please remove the old one and install the new one with the code shown in the tab section. Follow the instructions for SDK and CLI as shown here:

### Code versions

# [Azure CLI](#tab/cli)

```cli
az extension remove -n ml
az extension add -n ml --yes
az extension show -n ml #(the version value needs to be 2.15.1 or later)
```

# [Python SDK](#tab/python)

```python
pip uninstall azure-ai-ml
pip install azure-ai-ml
pip show azure-ai-ml #(the version value needs to be 1.5.0 or later)
```

# [Studio](#tab/azure-studio)

Not available.

---

## Create a Snowflake DB connection

# [Azure CLI](#tab/cli)
This YAML file creates a Snowflake DB connection. Be sure to update the appropriate values:

```yaml
# my_snowflakedb_connection.yaml
$schema: http://azureml/sdk-2-0/Connection.json
type: snowflake
name: my-sf-db-connection # add your datastore name here

target: jdbc:snowflake://<myaccount>.snowflakecomputing.com/?db=<mydb>&warehouse=<mywarehouse>&role=<myrole>
# add the Snowflake account, database, warehouse name and role name here. If no role name provided it will default to PUBLIC
credentials:
    type: username_password
    username: <username> # add the Snowflake database user name here or leave this blank and type in CLI command line
    password: <password> # add the Snowflake database password here or leave this blank and type in CLI command line
```

Create the Azure Machine Learning connection in the CLI:

### Option 1: Use the username and password in YAML file

```azurecli
az ml connection create --file my_snowflakedb_connection.yaml
```

### Option 2: Override the username and password at the command line

```azurecli
az ml connection create --file my_snowflakedb_connection.yaml --set credentials.username="XXXXX" credentials.password="XXXXX"
```

# [Python SDK](#tab/python)

### Option 1: Load connection from YAML file

```python
from azure.ai.ml import MLClient, load_workspace_connection

ml_client = MLClient.from_config()

wps_connection = load_workspace_connection(source="./my_snowflakedb_connection.yaml")
wps_connection.credentials.username="XXXXX"
wps_connection.credentials.password="XXXXXXXX"
ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

### Option 2: Use WorkspaceConnection() in a Python script

```python
from azure.ai.ml import MLClient
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration

target= "jdbc:snowflake://<myaccount>.snowflakecomputing.com/?db=<mydb>&warehouse=<mywarehouse>&role=<myrole>"
# add the Snowflake account, database, warehouse name and role name here. If no role name provided it will default to PUBLIC
name= <my_snowflake_connection> # name of the connection
wps_connection = WorkspaceConnection(name= name,
type="snowflake",
target= target,
credentials= UsernamePasswordConfiguration(username="XXXXX", password="XXXXXX")
)

ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

# [Studio](#tab/azure-studio)

1. Navigate to the [Azure Machine Learning studio](https://ml.azure.com).

1. Under **Assets** in the left navigation, select **Data**. Next, select the **Data Connection** tab. Then select Create as shown in this screenshot:

   :::image type="content" source="media/how-to-connection/create-new-data-connection.png" lightbox="media/how-to-connection/create-new-data-connection.png" alt-text="Screenshot showing the start of a new data connection in Azure Machine Learning studio UI.":::

1. In the **Create connection** pane, fill in the values as shown in the screenshot. Choose Snowflake for the category, and **Username password** for the Authentication type. Be sure to specify the **Target** textbox value in this format, filling in your specific values between the < > characters:

   **jdbc:snowflake://\<myaccount\>.snowflakecomputing.com/?db=\<mydb\>&warehouse=\<mywarehouse\>&role=\<myrole\>**

   :::image type="content" source="media/how-to-connection/create-snowflake-connection.png" lightbox="media/how-to-connection/create-snowflake-connection.png" alt-text="Screenshot showing creation of a new Snowflake connection in Azure Machine Learning studio UI.":::

1. Select Save to securely store the credentials in the key vault associated with the relevant workspace. This connection is used when running a data import job.

---

## Create an Azure SQL DB connection

# [Azure CLI](#tab/cli)

This YAML script creates an Azure SQL DB connection. Be sure to update the appropriate values:

```yaml
# my_sqldb_connection.yaml
$schema: http://azureml/sdk-2-0/Connection.json

type: azure_sql_db
name: my-sqldb-connection

target: Server=tcp:<myservername>,<port>;Database=<mydatabase>;Trusted_Connection=False;Encrypt=True;Connection Timeout=30
# add the sql servername, port addresss and database
credentials:
    type: sql_auth
    username: <username> # add the sql database user name here or leave this blank and type in CLI command line
    password: <password> # add the sql database password here or leave this blank and type in CLI command line
```

Create the Azure Machine Learning connection in the CLI:

### Option 1: Use the username / password from YAML file

```azurecli
az ml connection create --file my_sqldb_connection.yaml
```

### Option 2: Override the username and password in YAML file

```azurecli
az ml connection create --file my_sqldb_connection.yaml --set credentials.username="XXXXX" credentials.password="XXXXX"
```

# [Python SDK](#tab/python)

### Option 1: Load connection from YAML file

```python
from azure.ai.ml import MLClient, load_workspace_connection

ml_client = MLClient.from_config()

wps_connection = load_workspace_connection(source="./my_sqldb_connection.yaml")
wps_connection.credentials.username="XXXXXX"
wps_connection.credentials.password="XXXXXxXXX"
ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

### Option 2: Using WorkspaceConnection()

```python
from azure.ai.ml import MLClient
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration

target= "Server=tcp:<myservername>,<port>;Database=<mydatabase>;Trusted_Connection=False;Encrypt=True;Connection Timeout=30"
# add the sql servername, port address and database

name= <my_sql_connection> # name of the connection
wps_connection = WorkspaceConnection(name= name,
type="azure_sql_db",
target= target,
credentials= UsernamePasswordConfiguration(username="XXXXX", password="XXXXXX")
)

ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

# [Studio](#tab/azure-studio)

1. Navigate to the [Azure Machine Learning studio](https://ml.azure.com).

1. Under **Assets** in the left navigation, select **Data**. Next, select the  **Data Connection**  tab. Then select Create as shown in this screenshot:

   :::image type="content" source="media/how-to-connection/create-new-data-connection.png" lightbox="media/how-to-connection/create-new-data-connection.png" alt-text="Screenshot showing the start of a new data connection in Azure Machine Learning studio UI.":::

1. In the **Create connection** pane, fill in the values as shown in the screenshot. Choose AzureSqlDb for the category, and **Username password** for the Authentication type. Be sure to specify the **Target** textbox value in this format, filling in your specific values between the < > characters:

   **Server=tcp:\<myservername\>,\<port\>;Database=\<mydatabase\>;Trusted_Connection=False;Encrypt=True;Connection Timeout=30**

   :::image type="content" source="media/how-to-connection/how-to-create-azuredb-connection.png" lightbox="media/how-to-connection/how-to-create-azuredb-connection.png" alt-text="Screenshot showing creation of a new Azure DB connection in Azure Machine Learning studio UI.":::

---

## Create Amazon S3 connection

# [Azure CLI](#tab/cli)

Create an Amazon S3 connection with the following YAML file. Be sure to update the appropriate values:

```yaml
# my_s3_connection.yaml
$schema: http://azureml/sdk-2-0/Connection.json

type: s3
name: my_s3_connection

target: <mybucket> # add the s3 bucket details
credentials:
    type: access_key
    access_key_id: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX # add access key id
    secret_access_key: XxXxXxXXXXXXXxXxXxxXxxXXXXXXXXxXxxXXxXXXXXXXxxxXxXXxXXXXXxXXxXXXxXxXxxxXXxXXxXXXXXxXxxXX # add access key secret
```

Create the Azure Machine Learning connection in the CLI:

```azurecli
az ml connection create --file my_s3_connection.yaml
```

# [Python SDK](#tab/python)

### Option 1: Load connection from YAML file

```python
from azure.ai.ml import MLClient, load_workspace_connection

ml_client = MLClient.from_config()

wps_connection = load_workspace_connection(source="./my_s3_connection.yaml")
ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

### Option 2: Use WorkspaceConnection() in a Python script

```python
from azure.ai.ml import MLClient
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import AccessKeyConfiguration

target=<mybucket> # add the s3 bucket details
name=<my_s3_connection> # name of the connection
wps_connection=WorkspaceConnection(name=name,
type="s3",
target= target,
credentials= AccessKeyConfiguration(access_key_id="XXXXXX",acsecret_access_key="XXXXXXXX")
)

ml_client.connections.create_or_update(workspace_connection=wps_connection)

```

# [Studio](#tab/azure-studio)

1. Navigate to the [Azure Machine Learning studio](https://ml.azure.com).

1. Under **Assets** in the left navigation, select **Data**. Next, select the **Data Connection** tab. Then select Create as shown in this screenshot:

   :::image type="content" source="media/how-to-connection/create-new-data-connection.png" lightbox="media/how-to-connection/create-new-data-connection.png" alt-text="Screenshot showing the start of a new data connection in Azure Machine Learning studio UI.":::

1. In the **Create connection** pane, fill in the values as shown in the screenshot. Choose S3 for the category, and **Username password** for the Authentication type. Be sure to specify the **Target** textbox value in this format, filling in your specific values between the < > characters:

   **\<target\>**

   :::image type="content" source="media/how-to-connection/how-to-create-amazon-s3-connection.png" lightbox="media/how-to-connection/how-to-create-amazon-s3-connection.png" alt-text="Screenshot showing creation of a new Amazon S3 connection in Azure Machine Learning studio UI.":::

---

## Non-data connections

You can use these connection types to connect to Git:

- Python feed
- Azure Container Registry
- a connection that uses an API key

These connections aren't data connections, but are used to connect to external services for use in your code.

### Git

# [Azure CLI](#tab/cli)

Create a Git connection with one of following YAML file. Be sure to update the appropriate values:

* Connect using a personal access token (PAT):

   ```yml
   #Connection.yml
   name: test_ws_conn_git_pat
   type: git
   target: https://github.com/contoso/contosorepo
   credentials:
      type: pat
      pat: dummy_pat
   ```

* Connect to a public repo (no credentials):

   ```yml
   #Connection.yml

   name: git_no_cred_conn
   type: git
   target: https://https://github.com/contoso/contosorepo

   ```

Create the Azure Machine Learning connection in the CLI:

```azurecli
az ml connection create --file connection.yaml
```

# [Python SDK](#tab/python)

The following example creates a Git connection to a GitHub repo. A Personal Access Token (PAT) authenticates the connection:

```python
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration, PatTokenConfiguration


name = "my_git_conn"

target = "https://github.com/myaccount/myrepo"

wps_connection = WorkspaceConnection(
    name=name,
    type="git",
    target=target,
    credentials=PatTokenConfiguration(pat="XXXXXXXXX"),    
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
```

# [Studio](#tab/azure-studio)

You can't create a Git connection in studio.

---

### Python feed

# [Azure CLI](#tab/cli)

Create a connection to a Python feed with one of following YAML files. Be sure to update the appropriate values:

* Connect using a personal access token (PAT):

   ```yml
   #Connection.yml
   name: test_ws_conn_python_pat
   type: python_feed
   target: https://test-feed.com
   credentials:
      type: pat
      pat: dummy_pat
   ```

* Connect using a username and password:

   ```yml
   name: test_ws_conn_python_user_pass
   type: python_feed
   target: https://test-feed.com
   credentials:
      type: username_password
      username: john
      password: pass

   ```

* Connect to a public feed (no credentials):

   ```yml
   name: test_ws_conn_python_no_cred
   type: python_feed
   target: https://test-feed.com3
   ```

Create the Azure Machine Learning connection in the CLI:

```azurecli
az ml connection create --file connection.yaml
```

# [Python SDK](#tab/python)

The following example creates a Python feed connection. A Personal Access Token (PAT), or a user name and password, authenticates the connection:

```python
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration, ManagedIdentityConfiguration  


name = "my_pfeed_conn"

target = "https://XXXXXXXXX.core.windows.net/mycontainer"

wps_connection = WorkspaceConnection(
    name=name,
    type="python_feed",
    target=target,
    #credentials=UsernamePasswordConfiguration(username="xxxxx", password="xxxxx"), 
    credentials=PatTokenConfiguration(pat="XXXXXXXXX"),    

    #credentials=None
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
```

# [Studio](#tab/azure-studio)

You can't create a Python feed connection in studio.

---

### Azure Container Registry

# [Azure CLI](#tab/cli)

Create a connection to an Azure Container Registry with one of following YAML files. Be sure to update the appropriate values:

* Connect using Microsoft Entra ID authentication:

   ```yml
   name: test_ws_conn_cr_managed
   type: container_registry
   target: https://test-feed.com
   credentials:
      type: managed_identity
      client_id: client_id
      resource_id: resource_id
   ```

* Connect using a username and password:

   ```yml
   name: test_ws_conn_cr_user_pass
   type: container_registry
   target: https://test-feed.com2
   credentials:
      type: username_password
      username: contoso
      password: pass
   ```

Create the Azure Machine Learning connection in the CLI:

```azurecli
az ml connection create --file connection.yaml
```

# [Python SDK](#tab/python)

The following example creates an Azure Container Registry connection. A managed identity authenticates this connection:

```python
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration, PatTokenConfiguration  


name = "my_acr_conn"

target = "https://XXXXXXXXX.core.windows.net/mycontainer"

wps_connection = WorkspaceConnection(
    name=name,
    type="container_registry",
    target=target,
    credentials=ManagedIdentityConfiguration (client_id="xxxxx", resource_id="xxxxx"),    
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
```

# [Studio](#tab/azure-studio)

You can't create an Azure Container Registry connection in studio.

---

### API key

The following example creates an API key connection:

```python
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration, ApiKeyConfiguration


name = "my_api_key"

target = "https://XXXXXXXXX.core.windows.net/mycontainer"

wps_connection = WorkspaceConnection(
    name=name,
    type="apikey",
    target=target,
    credentials=ApiKeyConfiguration(key="XXXXXXXXX"),    
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
```

## Related content

If you use a data connection (Snowflake DB, Amazon S3, or Azure SQL DB), these articles offer more information:

- [Import data assets](how-to-import-data-assets.md)
- [Schedule data import jobs](how-to-schedule-data-import.md)