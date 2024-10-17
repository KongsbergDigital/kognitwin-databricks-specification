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


