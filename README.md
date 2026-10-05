# databricks-incremental-data-warehouse
End-to-end Data Warehouse built with Databricks, featuring staging layers, incremental MERGE-based loading, data transformation, testing, dimensional modeling, and Power BI analytics.

## 📌 Overview

This project is an end-to-end **Data Warehouse and Analytics solution** built using **Databricks SQL**.

The project processes data from multiple CRM and ERP source datasets and transforms it into a structured analytical model that can be consumed by **Power BI**.

The main focus of the project is not simply moving data from one layer to another, but implementing a reliable **incremental data processing strategy** using temporary staging layers, permanent historical layers, data transformation, deduplication, and `MERGE` operations.

The warehouse follows a **Bronze, Silver, and Gold** layered architecture, with staging areas used to isolate incoming batches before they are compared with the permanent data.

---

# 🎯 Project Objectives

The project was built to practice and demonstrate the following Data Engineering concepts:

- Incremental data loading.
- Temporary staging layers.
- Historical data preservation.
- Insert, update, and skip logic.
- Data cleaning and transformation.
- Deduplication.
- Business-key based record matching.
- Data quality validation.
- Dimensional modeling.
- Separation between raw, cleaned, and analytical data.
- Testing different ingestion scenarios.
- Preparing data for BI consumption.

A major design goal was to avoid rebuilding the warehouse from scratch every time new data arrives.

Instead, each incoming batch is treated as a new set of data that must be compared against the existing warehouse state.

---

# 🧰 Technologies

- **Databricks**
- **Databricks SQL**
- **Apache Spark**
- **Delta Lake**
- **SQL**
- **Power BI**
- **Git / GitHub**

---

# 📂 Source Data

The project uses six CSV datasets originating from two source systems.

### CRM

- `crm_cust_info`
- `crm_prd_info`
- `crm_sales_details`

### ERP

- `erp_cust_az12`
- `erp_loc_a101`
- `erp_px_cat_g1v2`

The datasets are manually uploaded into Databricks because the project is implemented using Databricks Free Edition.

The original datasets are available in:

`datasets/`

---

# 🔄 Incremental Loading Strategy

Incremental loading is one of the main concepts implemented in this project.

Instead of deleting and recreating the warehouse every time a new dataset arrives, the pipeline compares the incoming data with the existing permanent layer.

Each incoming batch follows the general process:

1. Load the incoming batch into a temporary staging table.
2. Compare the staging data with the permanent target table.
3. Identify new records.
4. Identify existing records whose values have changed.
5. Ignore records that are already identical.
6. Insert new records.
7. Update changed records.
8. Remove the temporary staging table after successful processing.

This approach allows the warehouse to retain historical data while processing only the changes introduced by new batches.

---

# 🧱 Why Staging Layers Are Used

Temporary staging layers are intentionally separated from the permanent Bronze and Silver tables.

The staging layer represents **the current incoming batch**, while the permanent layer represents **the accumulated warehouse data**.

For example:

```text
Incoming CSV
     ↓
Staging Table
     ↓
Compare with Permanent Table
     ↓
Insert / Update / Skip
     ↓
Permanent Table
     ↓
Drop Staging Table
```

This separation provides several advantages:

- The incoming batch can be inspected before modifying permanent data.
- New data can be compared against existing records.
- Permanent historical data is protected from direct replacement.
- The pipeline can process partial batches.
- Temporary ingestion data does not accumulate indefinitely.
- The logic for ingestion and permanent storage remains clearly separated.

The staging tables therefore act as a **controlled boundary between incoming data and the warehouse**.

---

# 🥉 Bronze Layer

The Bronze layer is the permanent **historical raw-data layer**.

Data first enters temporary Bronze staging tables.

For example:

`bronze_staging.crm_cust_info`

is processed into:

`bronze.crm_customers`

The Bronze procedure compares the staging batch with the existing Bronze table using the appropriate business key.

### Bronze loading behavior

If the target table does not exist yet:

All incoming records are treated as new and inserted.

If the target table already exists:

- New records are inserted.
- Existing records with changed values are updated.
- Existing records with identical values are skipped.

This makes the Bronze layer capable of handling both the **first-ever ingestion** and subsequent incremental loads.

---

# 🆕 First Ingestion Handling

A special case occurs when the warehouse is completely empty.

There is no permanent Bronze table available for comparison.

Instead of requiring a separate manual initialization process, the Bronze procedure checks whether the target table exists.

If it does not exist:

- The staging data is considered new.
- The Bronze target table is created.
- All valid incoming records are inserted.

This allows the same ingestion procedure to support both:

- Initial warehouse creation.
- Subsequent incremental ingestion.

---

# 🔁 Insert, Update, and Skip Logic

The incremental process is based on three possible outcomes for each incoming record.

### 1. New Record

The business key does not exist in the permanent table.

**Action:** Insert.

### 2. Changed Record

The business key already exists, but one or more values have changed.

**Action:** Update.

### 3. Unchanged Record

The business key exists and all relevant values are identical.

**Action:** Skip.

This prevents the pipeline from repeatedly rewriting records that have not changed.

`MERGE` operations are used to implement this logic.

---

# 🔑 Business Keys

The pipeline uses business keys to determine whether an incoming record represents an existing entity.

Examples include:

| Dataset | Business Key |
|---|---|
| Customers | `cst_id` |
| Products | `prd_id` |
| Sales | `sls_ord_num + sls_prd_key` |
| ERP Customers | `CID` |
| ERP Locations | `CID` |
| Product Categories | `ID` |

The sales table requires a composite key because an order can contain multiple products.

Therefore, `sls_ord_num` alone is not sufficient to uniquely identify a sales line.

The combination of:

`order number + product number`

represents the individual sales record used by the pipeline.

---

# 🥈 Silver Layer

The Silver layer contains cleaned, standardized, and transformed historical data.

Silver processing follows the same staging principle used by Bronze.

Data is first transformed into temporary Silver staging tables.

For example:

`silver_staging.crm_customers`

is generated from:

`bronze.crm_customers`

The transformed staging data is then compared with:

`silver.crm_customers`

The pipeline again performs:

- Insert for new records.
- Update for changed records.
- Skip for unchanged records.

After successful processing, the temporary Silver staging tables are removed.

---

# 🧹 Silver Transformations

The Silver layer is responsible for improving the quality and consistency of the raw data.

Examples include:

### Customer data

- Trimming names.
- Standardizing marital status.
- Standardizing gender values.
- Removing invalid customer IDs.
- Handling duplicate customer records.

### Product data

- Extracting category identifiers.
- Standardizing product lines.
- Handling invalid product costs.
- Deriving product validity periods.

### Sales data

- Converting numeric dates into proper date values.
- Handling invalid dates.
- Validating sales amounts.
- Recalculating incorrect sales values.
- Correcting invalid prices.

### ERP customer data

- Removing source-specific ID prefixes.
- Validating birthdates.
- Standardizing gender values.

### ERP location data

- Normalizing customer IDs.
- Standardizing country values.

The purpose of the Silver layer is to provide a reliable and consistent representation of the source data before it reaches the analytical layer.

---

# 🧪 Deduplication

Deduplication is performed before records are merged into permanent layers when necessary.

This is particularly important for datasets where multiple source records may share the same business key.

For example, customer records can be ranked using window functions so that the appropriate record is selected before the data reaches the target table.

This prevents a `MERGE` operation from receiving multiple source rows for the same target key.

The sales dataset also required special consideration because an order can legitimately contain multiple product lines.

Therefore, the business key for sales is not simply the order number.

---

# 🥇 Gold Layer

The Gold layer provides the final analytical representation of the warehouse.

It is built from the cleaned Silver data and is designed for consumption by Power BI.

The Gold layer contains:

- `gold.dim_customers`
- `gold.dim_products`
- `gold.fact_sales`

The model follows dimensional modeling principles, separating descriptive information into dimensions and transactional information into a central fact table.

---

# 📊 Gold Layer as an Analytical Interface

Unlike Bronze and Silver, the Gold layer is implemented using views.

The views reference the current Silver tables, meaning that the analytical layer automatically reflects changes made to the underlying data.

The Gold layer therefore acts as a clean analytical interface between the warehouse and BI tools.

Power BI connects directly to these Gold views rather than consuming the raw or cleaned warehouse tables.

---

# 🧪 Testing Strategy

The pipeline was tested using multiple scenarios designed specifically around its incremental loading behavior.

### Test 1 — Empty Warehouse

The warehouse starts without existing permanent tables.

**Expected behavior:**

All incoming records are treated as new.

**Result:** ✅ Passed

### Test 2 — New Data Batch

A new batch containing additional records is processed.

**Expected behavior:**

Only new records are inserted.

**Result:** ✅ Passed

### Test 3 — Identical Data

The same batch is processed again without modifications.

**Expected behavior:**

Existing records are recognized as unchanged and skipped.

**Result:** ✅ Passed

### Test 4 — Changed Data

Existing records are modified in the incoming batch.

**Expected behavior:**

The corresponding permanent records are updated.

**Result:** ✅ Passed

### Test 5 — Duplicate Source Records

Source data containing duplicate business keys is tested.

**Expected behavior:**

Duplicates are handled before the `MERGE` operation so that multiple source records do not incorrectly match the same target record.

**Result:** ✅ Passed

### Test 6 — Partial Batch

Only some of the available source datasets are provided.

**Expected behavior:**

The available datasets are processed without requiring every source dataset to be present.

**Result:** ✅ Passed

The SQL test scripts are available in:

`tests/`

---

# 🔍 Data Quality

Data quality checks are performed as part of the transformation and testing process.

Examples include:

- NULL business keys.
- Duplicate business keys.
- Invalid dates.
- Invalid prices.
- Invalid sales amounts.
- Invalid quantities.
- Missing relationships between sales and customers.
- Missing relationships between sales and products.
- Inconsistent categorical values.

The purpose of these checks is to prevent poor-quality source data from propagating into the analytical layer.

---

# ⚙️ Pipeline Procedures

The project separates processing into procedures for each layer.

### Bronze Procedure

Responsible for:

- Reading Bronze staging tables.
- Comparing incoming data with permanent Bronze data.
- Inserting new records.
- Updating changed records.
- Skipping unchanged records.
- Removing Bronze staging tables.

### Silver Procedure

Responsible for:

- Reading Bronze data.
- Cleaning and transforming the data.
- Creating Silver staging tables.
- Comparing transformed data with permanent Silver data.
- Inserting, updating, and skipping records.
- Removing Silver staging tables.

### Gold Procedure

Responsible for:

- Creating or replacing the analytical Gold views.
- Building customer and product dimensions.
- Building the sales fact.
- Preparing the final model for Power BI.

The procedures are available in:

`sql/`

---

# 📁 Project Structure

```text
data-warehouse-project/
│
├── datasets/
│   ├── crm/
│   └── erp/
│
├── sql/
│   ├── bronze/
│   ├── silver/
│   ├── gold/
│   └── orchestration/
│
├── tests/
│
├── docs/
│   ├── architecture/
│   ├── data_model/
│   ├── transformations/
│   └── testing/
│
├── powerbi/
│
└── README.md
```

---

# 📊 Power BI

The Gold layer is connected to Power BI as the final analytical data source.

The dashboard is intended to provide insights into:

- Sales performance.
- Order volume.
- Product performance.
- Customer activity.
- Geographic distribution.
- Product categories.
- Sales trends.

Power BI screenshots and related documentation are available in:

`powerbi/`

---

# 🧠 Key Design Decisions

### Permanent layers vs. temporary staging

Permanent Bronze and Silver tables retain accumulated warehouse data, while staging tables represent only the current processing batch.

This prevents staging data from becoming a second permanent storage layer.

### Incremental loading instead of full reloads

The pipeline does not truncate and rebuild the warehouse for every batch.

Instead, incoming data is compared against existing records and only the required changes are applied.

### `MERGE` for incremental processing

`MERGE` provides a single mechanism for handling new and changed records while allowing unchanged records to remain untouched.

### Historical Bronze storage

Raw records are retained in Bronze to provide a historical source for auditing, troubleshooting, and future reprocessing.

### Separate transformation staging

Silver transformations are performed in a temporary staging layer before modifying the permanent Silver tables.

This keeps transformation logic separate from the final cleaned dataset.

### Business-key based matching

Records are matched using business keys rather than relying only on physical row positions or ingestion order.

### Gold views

The Gold layer provides a lightweight analytical interface over the Silver layer without creating unnecessary physical copies of the data.

---

# 🚀 Future Improvements

Possible future improvements include:

- Automated file ingestion.
- Automated scheduling and orchestration.
- Pipeline execution logging.
- More extensive automated data-quality validation.
- Centralized error handling and recovery.
- A dedicated Date dimension.
- Additional analytical metrics.
- CI/CD for SQL procedures and tests.
- Automated testing as part of pipeline execution.

---

# 📚 Documentation

Additional project documentation includes:

- Architecture documentation.
- Data model documentation.
- Transformation documentation.
- Testing documentation.
- SQL procedures.
- Dataset descriptions.
- Power BI dashboard documentation.

---

# 👤 Project

This project was developed as a personal **Data Engineering** project to practice designing and implementing an end-to-end data warehouse using Databricks.

The project places particular emphasis on **incremental data loading, temporary staging layers, historical data preservation, data transformation, business-key based matching, testing, and analytical modeling**.

Rather than using a full-refresh approach, the pipeline was designed around the idea that incoming data represents a new batch that must be compared with the existing warehouse state.

The project therefore focuses not only on **where the data goes**, but also on **how the pipeline decides what should happen to each incoming record**.
