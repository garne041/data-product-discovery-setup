# Data Product Discovery Setup

This project implements an AI-powered data product discovery system using Databricks Mosaic AI. It creates a RAG (Retrieval Augmented Generation) agent that helps business and data users discover relevant data products from a vector search index containing embeddings of data product metadata, schemas, and descriptions.

## Overview

The system enables users to:
* Query for data products using natural language
* Receive ranked, relevant data product recommendations
* Access structured metadata including table descriptions, schemas, and ownership information
* Discover relationships between data products and underlying tables

## Prerequisites

Before running the notebooks, ensure you have the following resources created:

### 1. Unity Catalog Resources

#### Catalogs
You'll need **two catalogs**:

1. **Data Catalog** (e.g., `main_jcg`):
   * Contains the actual data tables (loan acquisition, borrower demographics, etc.)
   * Schema: `fnma`
   * This is where the 15 FNMA mortgage-related tables will be created

2. **Product Catalog** (e.g., `fnma_product_catalog_jcg`):
   * Contains the metadata and product definitions
   * Schema: `default`
   * This is where the product catalog and mapping tables will be stored

**To create catalogs:**
```sql
CREATE CATALOG IF NOT EXISTS main_jcg;
CREATE CATALOG IF NOT EXISTS fnma_product_catalog_jcg;

CREATE SCHEMA IF NOT EXISTS main_jcg.fnma;
CREATE SCHEMA IF NOT EXISTS fnma_product_catalog_jcg.default;
```

#### Schemas with Product Definitions
In the **Product Catalog**, create schemas for each data product with descriptions and tags:

```sql
-- Example: Create a schema for each data product
CREATE SCHEMA IF NOT EXISTS fnma_product_catalog_jcg.acquisition_and_origination_analytics
COMMENT 'Data product for analyzing loan acquisition trends, originator performance, and borrower risk profiles at origination';

```

**Required Product Schemas:**
* `acquisition_and_origination_analytics`
* `investor_and_security_allocation`
* `fraud_detection_and_process_optimization`
* `compliance_and_regulatory_reporting`
* `loss_and_credit_risk_management`
* `loan_performance_and_servicing`
* `collateral_and_property_intelligence`

### 2. Vector Search Endpoint

You'll need a Vector Search endpoint. The notebooks will attempt to create `demo_vs_endpoint` or fall back to `dbdemos_vs_endpoint`.

**To create manually (optional):**
```python
from databricks.vector_search.client import VectorSearchClient

vs_client = VectorSearchClient()
vs_client.create_endpoint(
    name="demo_vs_endpoint",
    endpoint_type="STANDARD"
)
```

### 3. Model Serving Endpoints

Ensure you have access to the following Foundation Model endpoints:
* `databricks-gpt-5` (or `databricks-claude-sonnet-4-5`)
* `databricks-bge-large-en` (for embeddings)

### 4. Permissions

Ensure you have:
* `CREATE TABLE` permissions on both catalogs
* `CREATE SCHEMA` permissions on both catalogs
* `USE CATALOG` and `USE SCHEMA` permissions
* Access to create and manage Vector Search indexes
* Access to deploy Model Serving endpoints

## Notebook Execution Guide

**⚠️ Important: Execute the notebooks in numerical order (1 → 2 → 3 → 4)**

### Notebook 1: Fake Data and Product Generation

**Purpose:** Creates 15 synthetic FNMA mortgage-related tables with realistic data

**What it creates:**
* `main_jcg.fnma.loan_acquisition` - Loan origination details
* `main_jcg.fnma.loan_performance` - Monthly payment status and performance metrics
* `main_jcg.fnma.borrower_demographics` - Borrower demographic profiles
* `main_jcg.fnma.property_characteristics` - Property and collateral attributes
* `main_jcg.fnma.loan_servicing_table` - Servicer information and transfers
* `main_jcg.fnma.mortgage_note_table` - Mortgage note terms and conditions
* `main_jcg.fnma.loss_severity_table` - Loss amounts after liquidation
* `main_jcg.fnma.application_table` - Loan application data
* `main_jcg.fnma.repayment_table` - Month-by-month payment history
* `main_jcg.fnma.housing_goal_table` - Affordable housing compliance flags
* `main_jcg.fnma.uniform_closing_table` - Standardized closing data
* `main_jcg.fnma.property_appraisal_table` - Property appraisal values
* `main_jcg.fnma.mbs_table` - MBS pooling information
* `main_jcg.fnma.credit_history` - Borrower credit characteristics
* `main_jcg.fnma.unit_rentals` - Unit-level rental details

**Configuration:**
Update these variables at the top of the notebook:
```python
CATALOG = 'main_jcg'  # Your data catalog name
SCHEMA = 'fnma'       # Your schema name
```

**Runtime:** ~5-10 minutes

---

### Notebook 2: Consolidating Product Metadata

**Purpose:** Creates mapping tables that link data products to their underlying tables and consolidates metadata

**What it creates:**
* `fnma_product_catalog_jcg.default.tables_and_products` - Mapping of products to tables
* `main_jcg.fnma.products_tables_descriptions` - Products with table descriptions
* `fnma_product_catalog_jcg.default.product_catalog` - **Main catalog table** with:
  * Product names and descriptions
  * Associated tables and their descriptions (JSON format)
  * Tags for categorization
  * Unique identifiers

**Configuration:**
Update these variables:
```python
CATALOG = "fnma_product_catalog_jcg"  # Your product catalog name
SCHEMA = "default"
```

**Key Output:** The `product_catalog` table is the source for the vector search index

**Runtime:** ~2-3 minutes

---

### Notebook 3: Vector Search Setup

**Purpose:** Creates a Delta Sync vector search index on the product catalog

**What it creates:**
* Vector Search endpoint (if needed): `demo_vs_endpoint`
* Vector Search index: `fnma_product_catalog_jcg.default.product_catalog_vector_index`
  * Embeddings generated from the `Description` column
  * Continuous sync enabled with Change Data Feed
  * Uses `databricks-bge-large-en` embedding model

**Configuration:**
```python
CATALOG = "fnma_product_catalog_jcg"
SCHEMA = "default"
SOURCE_TABLE = f"{CATALOG}.{SCHEMA}.product_catalog"
INDEX_SHORT_NAME = "product_catalog_vector_index"
```

**Important Notes:**
* The index creation may take 5-10 minutes to complete
* The notebook waits for the index to be ready before completing
* Change Data Feed is enabled on the source table for continuous sync

**Runtime:** ~10-15 minutes (including index creation and sync)

---

### Notebook 4: Agent Creation and Testing

**Purpose:** Creates, tests, evaluates, and deploys a tool-calling RAG agent

**What it creates:**
* `agent.py` - Python file containing the agent implementation
* MLflow logged model with the agent
* Unity Catalog registered model: `fnma_product_catalog_jcg.default.demo_data_dicovery_rag`
* Model Serving endpoint for the agent

**Agent Capabilities:**
* Interprets natural language queries about data products
* Searches the vector index for relevant products
* Ranks results by relevance, completeness, freshness, and business value
* Returns structured JSON responses with top 3 recommendations

**Configuration:**
```python
LLM_ENDPOINT_NAME = "databricks-gpt-5"
INDEX_NAME = "fnma_product_catalog_jcg.default.product_catalog_vector_index"
catalog = "fnma_product_catalog_jcg"
schema = "default"
model_name = "demo_data_dicovery_rag"
```

**Execution Steps:**
1. Install dependencies and create `agent.py`
2. Test the agent locally with sample queries
3. Evaluate the agent using MLflow scorers
4. Log the agent as an MLflow model
5. Register the model to Unity Catalog
6. Deploy the agent to Model Serving

**⚠️ Deployment Wait Time:** After running the deployment cell, **wait 15 minutes** for the endpoint to be created and become ready

**Runtime:** ~20-30 minutes (including deployment wait time)

---

## Output Structure

### Tables Created

#### Data Tables (in `main_jcg.fnma`)
15 tables containing synthetic FNMA mortgage data with 700-1000 rows each

#### Metadata Tables (in `fnma_product_catalog_jcg.default`)

1. **`tables_and_products`**
   * Columns: `product`, `table_name`, `full_table_name`
   * Maps each data product to its constituent tables

2. **`products_tables_descriptions`** (in `main_jcg.fnma`)
   * Columns: `product`, `tables_and_descriptions` (array of structs)
   * Aggregates table descriptions by product

3. **`product_catalog`** ⭐ **Main Table**
   * Columns:
     * `unique_id` - Sequential identifier
     * `Product_Name` - Data product name
     * `Description` - Business description of the product
     * `TAG_NAME` - Tag category
     * `TAG_VALUE` - Tag value
     * `table_names` - JSON string of tables and descriptions
   * This is the source table for the vector search index

### Vector Search Index

**Name:** `fnma_product_catalog_jcg.default.product_catalog_vector_index`

**Configuration:**
* **Type:** Delta Sync (continuous)
* **Primary Key:** `unique_id`
* **Embedding Source:** `Description` column
* **Embedding Model:** `databricks-bge-large-en`
* **Metadata Columns:** `Product_Name`, `Description`, `table_names`, `TAG_NAME`, `TAG_VALUE`

### Agent Model

**Name:** `fnma_product_catalog_jcg.default.demo_data_dicovery_rag`

**Input Format:**
```json
{
  "input": [
    {
      "role": "user",
      "content": "find data about borrower demographics"
    }
  ]
}
```

**Output Format:**
```json
{
  "query_understanding": "User is looking for borrower demographic data",
  "search_query": "borrower demographics",
  "results": [
    {
      "rank": 1,
      "data_product_name": "acquisition_and_origination_analytics",
      "full_identifier": "acquisition_and_origination_analytics",
      "table_names": "[{\"table_name\":\"borrower_demographics\",\"description\":\"...\"}]",
      "description": "Contains borrower demographic profiles...",
      "completeness_score": "High"
    }
  ],
  "recommended_action": "Use the borrower_demographics table..."
}
```

## Testing the Agent

After deployment, test the agent using:

```python
from databricks import agents

# Query the deployed agent
response = agents.query(
    endpoint_name="demo_data_dicovery_rag",
    messages=[{"role": "user", "content": "find loan performance data"}]
)
print(response)
```

## Troubleshooting

### Common Issues

1. **Catalog/Schema doesn't exist**
   * Create the required catalogs and schemas before running Notebook 1

2. **Vector Search endpoint quota exceeded**
   * The notebooks will fall back to `dbdemos_vs_endpoint`
   * Or manually specify an existing endpoint

3. **Index sync taking too long**
   * Check the index status in the Vector Search UI
   * Ensure Change Data Feed is enabled on the source table

4. **Agent deployment fails**
   * Verify you have permissions to create Model Serving endpoints
   * Check that all dependent resources (LLM endpoint, vector index) are accessible

5. **Agent returns empty results**
   * Verify the vector index has completed initial sync
   * Check that the source table has data
   * Test the vector search index directly before testing the agent

## Additional Resources

* [Databricks Vector Search Documentation](https://docs.databricks.com/en/generative-ai/vector-search.html)
* [Mosaic AI Agent Framework](https://docs.databricks.com/en/generative-ai/agent-framework/index.html)
* [Unity Catalog Documentation](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)

## Support

For questions or issues, please contact the workspace administrator or refer to the Databricks documentation.
