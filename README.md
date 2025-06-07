# ADF-project-incremental-loading-through-azure-sql
Incremental Data Load from Azure SQL Source Table to Destination Table Using ADF

## Description
This project implements an Azure Data Factory (ADF) pipeline that performs **incremental loading** from a source Azure SQL table (`Orders`) to a destination Azure SQL table (`Orders_final`). The pipeline ensures that only **new rows inserted** after the latest timestamp in the destination are transferred, optimizing data movement and avoiding duplicates.

## Objective
Automate the incremental transfer of order records using a dynamic SQL query based on the latest `InsertTime` already present in the target table.

---

## Pipeline Activities

### 1. Lookup Activity (`Lookup1`)
- **Purpose**: Fetch the latest `InsertTime` from the `Orders_final` table to use as a watermark for incremental loading.
- **Query Used**:
  ```sql
  SELECT MAX(InsertTime) AS date1 FROM Orders_final
  ```
- **Dataset Used**: `AzureSqlTable1` (points to `Orders_final` table)
- **Output**: Provides a timestamp used to filter new records in the next step.

### 2. Copy Data Activity (`Copy data1`)
- **Purpose**: Copy records from the `Orders` table that have `InsertTime` greater than the value fetched by the Lookup activity.
- **Dynamic Query**:
  ```sql
  SELECT OrderID, FirstName, InsertTime 
  FROM Orders 
  WHERE InsertTime > '@{activity('Lookup1').output.firstRow.date1}'
  ```
- **Source Dataset**: `AzureSqlTable2` (points to `Orders`)
- **Sink Dataset**: `AzureSqlTable3` (points to `Orders_final`)
- **Sink Settings**:
  - Sink Type: `AzureSqlSink`
  - Write Behavior: `insert`
  - Use Table Lock: `false`
  - Translator: `TabularTranslator` with type conversion

---

## Key Concepts Used

### Lookup Activity
- Retrieves reference data during runtime.
- Used to fetch watermark value for incremental logic.

### Expression Language
- Used to dynamically inject values into SQL queries.
- Example: `@{activity('Lookup1').output.firstRow.date1}`

### Copy Activity
- Performs data movement from source to sink.
- Configured to run only after Lookup succeeds.

### AzureSqlSource
- Reads from Azure SQL using dynamic query.
- Provides filtered data based on `InsertTime`.

### AzureSqlSink
- Writes to Azure SQL table using insert behavior.
- Suitable for appending new rows.

### Incremental Loading
- Loads only new data that hasn’t been copied before.
- Reduces performance overhead and avoids data duplication.

### Tabular Translator
- Ensures schema compatibility between source and sink.
- Allows type conversion where necessary.

---

## How It Works

1. **Lookup1** activity runs a query on `Orders_final` to fetch the latest `InsertTime`.
2. **Copy data1** then selects records from `Orders` where `InsertTime` is greater than the value fetched in step 1.
3. These records are inserted into `Orders_final`.

---

## Requirements

- Azure Data Factory instance
- Azure SQL Database with:
  - `Orders` table (source)
  - `Orders_final` table (sink)
- Linked services:
  - `AzureSqlTable1` (for Lookup)
  - `AzureSqlTable2` (for source)
  - `AzureSqlTable3` (for sink)

---

## Example Use Case

This pipeline is ideal for scenarios like:
- Ingesting transactional data on a scheduled basis
- Building ETL pipelines with staging and incremental logic
- Appending only new data during warehouse refresh

---

## Notes

- Ensure both source and sink tables are present and accessible via linked services.
- `InsertTime` column must exist and be populated in both tables.
- You can optionally handle null results in Lookup using default fallback expressions.
- Recommended to schedule this pipeline to run periodically (e.g., hourly or daily).
