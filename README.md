# Retail Sales Data Warehouse — ETL & Data Quality Validation

## Overview

This project builds an end-to-end Retail Sales Data Warehouse on the cloud using AWS S3 and Databricks. It covers the complete ETL lifecycle from raw source file ingestion through data quality validation to final dimensional modelling, following the Medallion Architecture (Bronze → Silver → Gold).

The pipeline ingests CSV source files, applies data transformations and quality checks, loads clean data into dimension and fact tables, and manages archival of historical files automatically using a Python script.

---

## Problem Statement

A retail chain wants to consolidate its sales data into a centralized Data Warehouse covering Customers, Products, Stores, and Daily Sales. The pipeline handles files arriving in an SFTP landing zone, moves previous files to an archival location when new files arrive, and loads clean validated data into dimension and fact tables supporting both full load and incremental load strategies.

---

## Tech Stack

- **Cloud Platform** — Amazon Web Services (AWS)
- **Object Storage** — Amazon S3
- **Identity & Access Management** — AWS IAM Role
- **Infrastructure Automation** — AWS CloudFormation
- **Data Platform** — Databricks
- **Query Language** — Databricks SQL
- **Scripting** — Python 3
- **Architecture** — Medallion Architecture (Bronze / Silver / Gold)
- **Modelling** — Star Schema with SCD Type 2

---

## Architecture

The project follows the Medallion Architecture with three layers inside Databricks connected to Amazon S3.

**Bronze Layer** ingests raw CSV files from S3 exactly as-is with no transformation. Four raw tables are created — customers_raw, products_raw, stores_raw, and sales_raw — each pointing to their respective S3 folder using Databricks read_files with pathGlobFilter so the table always picks up the latest file regardless of the timestamp in the filename.

**Silver Layer** applies all business transformations and data quality rules. This is where dirty data is rejected, nulls are handled, formats are standardised, and dimension and fact tables are populated. The silver layer uses MERGE statements so the pipeline is idempotent and safe to re-run.

**Gold Layer** holds the final clean dimensional model tables — DimCustomer, DimProduct, DimStore, and FactSales — which are ready for reporting and analysis.

---

## S3 Zone Design

The S3 bucket is divided into four zones that represent the lifecycle of every file.

The **SFTP zone** is the receiving gate where source CSV files arrive first. No transformation happens here.

The **Raw zone** holds a backup copy of the original files exactly as received. If anything goes wrong downstream, this zone preserves the original data.

The **Processed zone** holds files after ETL transformation is complete. This represents the production-ready version of the data.

The **Archive zone** stores all older versions of files across all three zones. When a new file arrives in any zone, the Python archival script automatically moves the previous file into the corresponding archive subfolder, ensuring only the latest file stays in each active zone at all times.

---

## Data Model

**DimCustomer** implements SCD Type 2. It tracks CustomerID, CustomerName, Email, City, and Address. When a customer's City or Address changes, the old record is expired by setting IsActive to 0 and EndDate to the load date, and a new active record is inserted with IsActive = 1 and EndDate = 9999-12-31. This preserves full history of customer changes.

**DimProduct** stores ProductID, ProductName, Category, and UnitPrice. No SCD is applied since prices are treated as static for this project. Records with UnitPrice = 0 are rejected during the silver layer load.

**DimStore** stores StoreID, StoreName, and Region. StoreName is standardised to Title Case and NULL Region values are defaulted to 'Unknown'.

**FactSales** stores one row per transaction with surrogate keys linking to all three dimensions. The Amount column is a derived field calculated as Quantity multiplied by UnitPrice, rounded to 2 decimal places. Records with Quantity = 0, duplicate TransactionIDs, and CustomerIDs not found in DimCustomer are all rejected before load.

---

## Data Quality Issues Handled

The source files contain intentional data quality problems that the silver layer detects and handles correctly.

In the customer file, the same CustomerID appears twice with an updated City or Address, email values are stored in uppercase, and City fields contain leading or trailing spaces. In the product file, some ProductName values have extra spaces and some records have UnitPrice equal to zero. In the store file, Region is missing for some stores and StoreName has inconsistent casing. In the sales file, some TransactionIDs are duplicated, some CustomerIDs do not exist in the customer file, some records have Quantity equal to zero, and TxnDate values use inconsistent date formats.

The silver layer applies TRIM, LOWER, and INITCAP transformations, rejects invalid records, standardises date formats to YYYY-MM-DD, and excludes orphan records that fail referential integrity checks.

---

## Archival Process

The archival.py Python script runs before each pipeline execution. It scans every active zone folder, lists all CSV files present, parses the date and time from each filename using the DDMMYYYY_HHMMSS naming convention, identifies the newest file by timestamp, copies all older files to the corresponding archive subfolder, and deletes them from the active zone. The script logs every action clearly, prints which file was kept and which were archived, and reports the total count of archived files at the end. If no old files are present, the script skips the folder cleanly without error.

---

## Load Strategy

**Day 1** performs a full load. All source records are loaded into the bronze tables and the silver MERGE inserts fresh records into all dimension and fact tables with IsActive = 1 for all customers.

**Day 2 onwards** performs incremental loads. The archival script runs first to move old files to archive. Bronze tables are recreated pointing to the new files. The silver MERGE compares incoming records against existing ones — unchanged records are skipped, changed customer records trigger the SCD2 expire-and-insert logic, and new transactions are appended to FactSales. The pipeline is designed to be idempotent, meaning running it twice on the same data produces the same result with no duplicate records.

---

## Validation Testing

30 SQL validation test cases are written across 6 sections covering every aspect of the pipeline.

**Source to Target Testing** validates that row counts match between source and silver, column mappings are correct, and data types are as expected on Amount, StartDate, and TxnDate columns.

**Transformation Testing** validates that all Email values are lowercase, CustomerName values are in Proper Case, City and Address fields are trimmed, NULL Regions are defaulted to Unknown, and Amount equals the correct rounded calculation of Quantity multiplied by UnitPrice.

**Data Quality Testing** validates that zero-price products are excluded, duplicate TransactionIDs are not present, no CustomerID has more than one active record, no NULL CustomerSK exists in FactSales, and all foreign keys in FactSales resolve correctly in their respective dimension tables.

**SCD Type 2 Testing** validates that expired records have a real EndDate and not 9999-12-31, active records always have EndDate of 9999-12-31, no overlapping date periods exist for the same CustomerID, and FactSales only links to active CustomerSK records.

**Incremental Load Testing** validates that Day 1 loads all customers as active, Day 2 correctly creates two rows for changed customers, unchanged customers are not duplicated, only new transactions are added to FactSales, and re-running the pipeline does not change any row counts.

**Archival Testing** validates that old files are correctly moved to archive when new files arrive, only one file remains in each active folder after archival, the latest file is correctly identified by its timestamp, archival is applied consistently across all zones, and the script logs the correct count of archived files.

---

## Deliverables

- ETL Test Plan
- Test Cases from Mapping Document (30 test cases across 6 sections)
- SQL Validation Queries
- Data Quality Issues List
- Defect Log with Severity
- Test Execution Summary
- Test Closure Report
- Archival Process Implementation with logging
- Monitoring and alerting for pipeline failures

---

## Key Concepts Demonstrated

- Medallion Architecture (Bronze / Silver / Gold) on Databricks
- SCD Type 2 implementation using MERGE statements
- Dynamic file ingestion using pathGlobFilter without hardcoded filenames
- Automated file lifecycle management across S3 zones using Python
- End-to-end ETL testing covering source-to-target, transformation, data quality, SCD2, incremental load, and archival validation
- Idempotent pipeline design using MERGE so re-runs are safe
- Star schema dimensional modelling with surrogate keys and referential integrity
