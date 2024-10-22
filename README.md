# Kognitwin - Databricks integration

This document explains the technical integration between Kognitwin and Databricks. The integration between Databricks and Kognitwin offers a comprehensive digital twin solution, 
with Databricks providing the heavy lifting for data processing, machine learning, and analytics, 
and Kognitwin offering an intuitive interface for visualization, interaction, simulation, and 
operational management. This combination ensures that businesses can optimize their physical 
assets in real-time while leveraging advanced data analytics to drive efficiency, reduce 
downtime, and enhance decision-making.

By using this architecture, companies can unlock the full potential of digital twins, driving 
operational improvements and business outcomes in various industries such as manufacturing, 
energy, and infrastructure

## Overview
There are two different integration methods from Kognitwin to Databricks:

1. **Data on demand**: Execute SQL statements directly on the Databricks' endpoints to load data on demand. The data is not stored in Kognitwin but can be displayed in a digital twin to enhance decision making. This method has limited contextualization possibilities. 

2. **Data ingestion**: Ingesting data from Databricks to Kognitwin can either be done from SQL Warehouses or Delta Sharing. The ingested data is stored in Kognitwin and can be transformed and contextualized to suit the requirements of a digital twin. 

## Data on demand
Data in Kognitwin is served from the `GET /assets` endpoint. To setup an integration to Databricks on this endpoint a source is required. The source must be of type `RemoteWebApi` and contain at least one endpoint definition.

```json
{
  "id": "<insert id>",
  "name": "<insert name>",
  "type": "RemoteWebApi",
  "server": {
    "url": "<insert url to Databricks endpoint>",
    "endpoints": [
      {
        "path": "/sql/statements",
        "headers": {
          "Content-Type": "application/json",
          "User-Agent": "Kognitwin",
        },
        "token": "<insert token>"
        "assetItems": "$body",
        "method": "post",
        "body": {
          "statement": "<insert SQL statement>",
          "warehouse_id": "<insert SQL warehouse id>",
          "wait_timeout": "50s"
        },
        "stringifyBody": true
      }
    ]
  }
}
```
The data in Databricks can then be queried by HTTP `GET /assets?source=<id>`.

This can also be used for browsing the Unity Catalog.
Example:
```json
{
  "id": "<insert id>",
  "name": "Databricks catalog",
  "type": "RemoteWebApi",
  "server": {
    "url": "https://<databricks-workspace>.azuredatabricks.net/api/2.0/unity-catalog",
    "endpoints": [
      {
        "path": "/",
        "query": {
          "type": "listCatalogs"
        },
        "token": "<insert token>",
        "headers": {
          "Content-Type": "application/json",
          "User-Agent": "Kognitwin"
        },
        "transform": [
            {
               "condition": "eq($query.type,listCatalogs)",
               "query": "catalogs"
            },
            {
               "condition": "eq($query.type,listSchemas)",
               "query": "schemas?catalog_name=$(query.catalog)"
            },
            {
               "condition": "eq($query.type,listTables)",
               "query": "tables?catalog_name=$(query.catalog)&schema_name=$(query.schema)"
            }
        ],
        "assetItems": "$body"
      }
     ]
  }
}
```

## Data ingestion
Kognitwin has functionality to ingest data from Delta Shares or SQL Warehouses in Databricks. This is achieved by utilizing the orchestration engine in Kognitwin. For the sake of simplicity, examples of manual ingestion are provided here, but other means or triggers are available in Kognitwin. To trigger a manual ingestion, do a HTTP `POST /tasks/queue` with a payload. The payload depends on whether to import data from Delta Share or SQL Warehouse.

### Delta Share
Ingest data from a Delta Share by using the `ImportDeltaSharingTask` task with the following payload. The `transform` property should be adjusted according to the data in Databricks but must define `id` and `source`.
```json
{
  "task": "ImportDeltaSharingTask",
  "id": "<insert id>",
  "configuration": {
    "batchSize": 1000,
    "profileFilePath": "/tmp/profile.share",
    "source": "<insert source id>",
    "delta_sharing_url": "<insert endpoint from share profile>",
    "token": "<insert token from share profile>",
    "share": "<insert share name>",
    "schema": "<insert schema name>",
    "table": "<insert table name>",
    "transform": {
      "id": "$row.id",
      "source": "<insert source id>",
      "data": "$row"
    }
  }
}
```

### SQL Warehouse
To import data from a SQL Warehouse, use the `importDatabricks` task with the following payload. The `transform` property should be adjusted according to the data in Databricks but must define `id` and `source`.
```json
{
  "task": "importDatabricks",
  "configuration": {
    "transform": {
      "id": "$item.id",
      "source": "<insert source id>",
      "data": "$item"
    },
    "databricks": {
      "token": "<insert token>",
      "host": "<insert host>",
      "path": "<insert path>",
      "sqlStatement": "<insert sql statement>"      
    }
  }
}
```

### Push interface
Whenever it is possible, a preferred solution is to push new/updated data to Kognitwin. This is done using cloudevents, where the cloudevent type is predefined and configured in Kognitwin. The configuration in Kognitwin will include a tranform from the cloudevent into the asset format used by Kognitwin. A simplified example config is shown here:
```json
{
	"id": "db:messages:examples-test1",
	"name": "Databricks example test message 1",
	"type": "messages",
	"messagesImport": {
		"messageType": "com.databricks.examples.test1",
		"assetItems": "$message.data",
		"transform": [
			{
				"id": "$item.itemId",
				"source": "db:$(item.site):example1",
				"type": "TestMessage",
				"name": "$item.name",
				"data": "$item",
				"meta": {
					"lastUpdated": "$date",
					"createdBy": "$identity",
					"system": "Databricks"
				},
				"derived": {
					"site": "$item.site"
				}
			}
		],
		"postOptions": {}
	}
}
```
Whith this configuration, a cloudevent example could look like this:
```json
{
  "id": "d8489e61-6989-48d8-b9b3-a0df5655cf33",
  "time": "2024-10-22T09:42:45.145Z",
  "type": "com.databricks.examples.test1",
  "source": "Databricks source1",
  "specversion": "1.0",
  "datacontenttype": "application/json",
  "schemaversion": "1.0",
  "data": [
    {
      "itemId": "1",
      "name": "test record 1",
      "site": "site1"
    },
    {
      "itemId": "2",
      "name": "test record 2",
      "site": "site2"
    }
  ]
}
```

To generate data for this, a databricks notebook can be used. A python script send the cloud event is shown below. In this example, the payload is hardcoded, but in the real integration the data payload would come from any of the data sources available within databricks or databricks' connected sources.
```python
from cloudevents.http import CloudEvent
from cloudevents.conversion import to_structured
import requests
import json
import uuid
from datetime import datetime

client_id = '<insert servive principal id>'
client_secret = dbutils.secrets.get(scope='<insert secret scope>', key='<insert secret key>')
token_url = '<insert authorization url>'

token_data = {
    'grant_type': 'client_credentials',
    'client_id': client_id,
    'client_secret': client_secret,
    'scope': 'openid offline_access'
}

token_response = requests.post(token_url, data=token_data)
token_response.raise_for_status()
access_token = token_response.json().get('access_token')

attributes = {
    "id": str(uuid.uuid4()),
    "time": datetime.utcnow().isoformat() + "Z",
    "type": "com.databricks.examples.test1",
    "source": "Databricks source1",
    "specversion": "1.0",
    "datacontenttype": "application/json",
    "schemaversion": "1.0"
}

data = [
    {
        "itemId": "1",
        "name": "test record 1",
        "site": "site1"
    },
    {
        "itemId": "2",
        "name": "test record 2",
        "site": "site2"
    }
]

event = CloudEvent(attributes, data)

headers, body = to_structured(event)
headers['Authorization'] = f'Bearer {access_token}'
headers['User-Agent'] = 'Databricks'

response = requests.post("https://<insert Kognitwin host>/api/messages", headers=headers, data=body)

print(f"Response status: {response.status_code}")
print(f"Response body: {response.text}")
```

The data created in Kognitwin will look like this:
GET {{baseUrl}}/assets?source=db:site1:example1,db:site2:example1
```json
[
    {
        "id": "1",
        "source": "db:site1:example1",
        "type": "TestMessage",
        "name": "test record 1",
        "data": {
            "itemId": "1",
            "name": "test record 1",
            "site": "site1"
        },
        "meta": {
            "lastUpdated": "2024-10-22T08:25:21.212Z",
            "createdBy": "messagesApi",
            "system": "Databricks"
        },
        "derived": {
            "site": "site1"
        }
    },
    {
        "id": "2",
        "source": "db:site2:example1",
        "type": "TestMessage",
        "name": "test record 2",
        "data": {
            "itemId": "2",
            "name": "test record 2",
            "site": "site2"
        },
        "meta": {
            "lastUpdated": "2024-10-22T09:42:46.526Z",
            "createdBy": "messagesApi",
            "system": "Databricks"
        },
        "derived": {
            "site": "site2"
        }
    }
]
```


