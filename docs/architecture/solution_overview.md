# Banking Data Platform - Solution Overview

## Project Purpose

This project simulates a production-style banking data platform for a fictional UK retail bank.

The platform will integrate customer, account, transaction, card, payment, KYC, merchant, branch and reference data from multiple source systems into a governed analytical lakehouse.

All data used in this project is synthetic. No real customer or bank data is used.

## Core Architecture

Source systems:

- SQL Server - core banking data
- REST API - customer and KYC data
- AWS S3 - card transaction data
- SharePoint - reference data
- Azure Blob Storage - payment and settlement files

Data platform:

- Azure Data Factory for ingestion and orchestration
- Azure Data Lake Storage Gen2 for landing/storage
- Azure Databricks for data engineering
- Delta Lake for Bronze, Silver and Gold tables
- PySpark and Spark SQL for transformation
- dbt for SQL-centric Gold modelling
- Unity Catalog for governance and access control
- GitHub and CI/CD for source control and deployment

## Engineering Principles

The platform will follow:

- metadata-driven ingestion
- incremental loading
- CDC where appropriate
- watermarking
- idempotent processing
- Bronze, Silver and Gold architecture
- data quality controls
- financial reconciliation
- audit logging
- observability
- PII governance
- least-privilege access
- CI/CD
- cost-conscious Azure PAYG usage

## Cost Principle

The design will follow production-level engineering practices while using portfolio-scale infrastructure.

Compute should run only when required.

Large datasets, continuous streaming, oversized clusters and unnecessary always-on services should be avoided unless required for a specific engineering test.